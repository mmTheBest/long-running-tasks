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

LOCK_FILE=/tmp/lrt-orchestrator.lock
PID_FILE=/tmp/lrt-worker.pid
LAST_COMMIT_FILE=/tmp/lrt-last-commit

Step 1 — Acquire lock:
  If $LOCK_FILE exists and the PID inside it is alive (kill -0), exit immediately with "orchestrator already running".
  Otherwise write your PID to $LOCK_FILE.
  Ensure lock is released on exit (trap).

Step 2 — Check for running worker:
  If $PID_FILE exists and the PID inside is alive:
    - Get latest commit: cd [PROJECT_PATH] && git log --oneline -1
    - Get commit age: git log -1 --format=%ct HEAD
    - If commit is < 30 minutes old: report one-line status, release lock, exit.
    - If commit is >= 30 minutes old: kill the PID, remove $PID_FILE, continue to Step 3.
  If $PID_FILE exists but PID is dead: remove $PID_FILE, continue to Step 3.

Step 3 — Check pause:
  If [PROJECT_PATH]/.pause exists: report "paused", release lock, exit.

Step 4 — Find next task:
  Read [PROJECT_PATH]/TODO.md
  Find the first line matching "- [ ]" that does NOT contain "BLOCKED:".
  If no unchecked tasks remain: report "all tasks complete — consider disabling this cron", release lock, exit.

Step 5 — Spawn worker:
  Run:
    cd [PROJECT_PATH] && nohup [AGENT_COMMAND] '[TASK_PROMPT]' > /tmp/lrt-worker.log 2>&1 &
    echo $! > $PID_FILE
    sleep 2 && kill -0 $(cat $PID_FILE) 2>/dev/null && echo "worker started" || echo "WARNING: worker failed to start"

  The task prompt must include:
  - "Read [CONTEXT_FILE] and TODO.md for project context."
  - The specific task description copied from TODO.md
  - "Run tests before committing. Fix failures before proceeding."
  - "Check off the completed item in TODO.md."
  - "Commit and push using the project's commit convention."
  - "Remove /tmp/lrt-worker.pid when finished."
  - "Run: openclaw system event --text 'Done: [BRIEF_SUMMARY]' --mode now"

Step 6 — Report:
  Compare latest commit hash with $LAST_COMMIT_FILE.
  If different: report task spawned + latest commit + diff stats. Update $LAST_COMMIT_FILE.
  If same: report task spawned + "no new commits since last check".

Step 7 — Release lock:
  Remove $LOCK_FILE.

CRITICAL RULES:
- Always spawn work if no worker is running and tasks remain. Never just report.
- Always release the lock before exiting, even on error.
- Keep reports under 80 words.
```

## Agent Command Examples

Replace `[AGENT_COMMAND]` with whichever coding agent is available:

```bash
# Claude Code
claude --dangerously-skip-permissions

# Codex
codex --yolo exec

# Any other agent that accepts a prompt argument
<agent-binary> <flags>
```

Choose based on your needs: agents with network access for tasks requiring git push or API calls; sandboxed agents for pure implementation work.

## Shutdown

The orchestrator stops spawning when all TODO.md items are checked. To stop earlier:

```bash
touch [PROJECT_PATH]/.pause     # pause — orchestrator skips spawning
rm [PROJECT_PATH]/.pause        # resume
```

Or disable the cron job via the OpenClaw `cron` tool with `action: "update"` and `patch: { "enabled": false }`.
