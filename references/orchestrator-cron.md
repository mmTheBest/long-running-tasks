# Orchestrator Cron Setup

## Cron Tool Invocation

Call the OpenClaw `cron` tool with `action: "add"`:

```json
{
  "action": "add",
  "job": {
    "name": "[PROJECT] orchestrator",
    "schedule": { "kind": "cron", "expr": "*/15 * * * *", "tz": "[YOUR_TIMEZONE]" },
    "sessionTarget": "isolated",
    "payload": {
      "kind": "agentTurn",
      "model": "anthropic/claude-sonnet-4-5",
      "timeoutSeconds": 180,
      "message": "SEE PROMPT BELOW"
    },
    "delivery": {
      "mode": "announce",
      "channel": "[CHANNEL]",
      "to": "[TARGET]"
    }
  }
}
```

**Settings:**
- **Schedule:** `*/10` to `*/30` depending on urgency. Shorter = less dead time between tasks.
- **Model:** Use a cheap, fast model — the orchestrator does light coordination, not coding.
- **Timeout:** 180s minimum — needs time to spawn a worker and verify it started.
- **Delivery:** Set `channel` and `to` for progress reports. Omit for no reporting.

## Orchestrator Prompt Template

Paste this as the `message` value. Replace all `[PLACEHOLDERS]`:

```
You are a development orchestrator for [PROJECT_NAME] at [PROJECT_PATH].

LOCK_FILE=/tmp/lrt-[PROJECT_SLUG]-orchestrator.lock
PID_FILE=/tmp/lrt-[PROJECT_SLUG]-worker.pid
LAST_COMMIT_FILE=/tmp/lrt-[PROJECT_SLUG]-last-commit
STALL_THRESHOLD=60  # minutes — increase for data/ML tasks (90-120)

Step 1 — Acquire lock:
  If $LOCK_FILE exists:
    - Read the PID inside it.
    - If that PID is alive (kill -0 $PID 2>/dev/null succeeds): another orchestrator is running. Exit with "orchestrator already running".
    - If that PID is dead: the lock is stale. Remove $LOCK_FILE and continue.
  Write your own PID ($$) to $LOCK_FILE.
  Set a trap to remove $LOCK_FILE on exit: trap 'rm -f $LOCK_FILE' EXIT

Step 2 — Check for running worker:
  If $PID_FILE exists:
    - Read the PID inside it.
    - If PID is alive:
      - Get latest commit: cd [PROJECT_PATH] && git log --oneline -1
      - Get commit timestamp: git log -1 --format=%ct HEAD
      - Calculate commit age in minutes.
      - If commit age < $STALL_THRESHOLD: report one-line status ("worker active, last commit: <hash> <age>min ago"), exit.
      - If commit age >= $STALL_THRESHOLD, check for file activity:
        - Count recently modified output files: find [PROJECT_PATH] -newer $LAST_COMMIT_FILE -type f -not -path '*/.git/*' -not -path '*/.venv/*' -not -path '*/node_modules/*' 2>/dev/null | wc -l
        - Check process CPU usage: ps -o cputime= -p $PID 2>/dev/null
        - If recent files exist OR CPU time is growing: report "worker active (no commit in <age>min but producing output files)", exit.
        - If NO recent files AND no CPU activity: worker is stalled. Kill it (kill $PID), remove $PID_FILE. Include "killed stalled worker (<age>min idle, no file activity)" in report. Continue to Step 3.
    - If PID is dead: remove $PID_FILE. Continue to Step 3.

Step 3 — Check pause:
  If [PROJECT_PATH]/.pause exists: report "paused — .pause file present", exit.

Step 4 — Find next task:
  Read [PROJECT_PATH]/TODO.md
  Find the first line matching "- [ ]" that does NOT contain "BLOCKED:".
  If no unchecked non-blocked tasks remain: report "all tasks complete — consider disabling this cron", exit.

Step 5 — Spawn worker:
  Run:
    cd [PROJECT_PATH] && nohup [AGENT_COMMAND] '[TASK_PROMPT]' > /tmp/lrt-[PROJECT_SLUG]-worker.log 2>&1 &
    echo $! > $PID_FILE
    sleep 2 && kill -0 $(cat $PID_FILE) 2>/dev/null && echo "worker verified" || echo "WARNING: worker failed to start"

  The task prompt must include:
  - "Read [CONTEXT_FILE] and TODO.md for project context."
  - The specific task description copied from TODO.md.
  - "IMPORTANT: Commit intermediate progress every 20-30 minutes. Do NOT wait until the task is fully done. The orchestrator monitors commit timestamps to detect stalls."
  - "Run tests before final commit. Fix failures before proceeding."
  - "Check off the completed item in TODO.md."
  - "Commit and push using the project's commit convention."
  - "Run: openclaw system event --text 'Done: [BRIEF_SUMMARY]' --mode now"

Step 6 — Report:
  Compare latest commit hash with $LAST_COMMIT_FILE.
  If different: report task spawned + commit hash + diff stats. Update $LAST_COMMIT_FILE.
  If same: report task spawned + "no new commits since last check".

CRITICAL RULES:
- Always spawn work if no worker is running and unchecked tasks remain. Never just report.
- Lock is auto-released by the EXIT trap. Do not skip the trap setup.
- Keep reports under 80 words.
- Use multi-signal stall detection (commit age + file activity + CPU). Commit age alone causes false kills on data-heavy tasks.
- Default stall threshold is 60 minutes. Use 90-120 for data/ML/download-heavy tasks.
```

## Stall Threshold Guidelines

The stall threshold determines how long a worker can go without a commit before being killed. Setting it too low causes a **kill-loop**: the orchestrator kills a working process, spawns a replacement, which also gets killed before it can finish.

| Workload | Recommended threshold |
|----------|-----------------------|
| Code edits, tests, linting | 30 min |
| Builds, API calls, installs | 45 min |
| Data downloads, file conversion | 60-90 min |
| ML training, heavy compute | 90-120 min |

**Always combine with file activity checks.** A worker writing output files to `data/`, `results/`, or `output/` is not stalled — even if it hasn't committed yet.

## Agent Command Examples

Replace `[AGENT_COMMAND]` with whichever coding agent is available:

```bash
# Claude Code (interactive approval — recommended for first runs)
claude

# Codex (sandboxed by default)
codex exec

# Any agent that accepts a prompt argument
<agent-binary> <flags>
```

> **Security note:** Some agents offer flags that bypass permission checks (e.g., `--dangerously-skip-permissions`, `--yolo`). Do not use these unless you have validated the worker prompts and TODO.md contents in a test repo first. Start with default (safe) mode and only relax permissions after you trust the pipeline.

## Shutdown

The orchestrator stops spawning when all TODO.md items are checked or blocked. To stop earlier:

```bash
touch [PROJECT_PATH]/.pause     # pause
rm [PROJECT_PATH]/.pause        # resume
```

Or disable the cron job: `cron` tool with `action: "edit"`, set `--disabled`.
