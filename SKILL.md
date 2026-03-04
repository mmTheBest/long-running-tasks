---
name: long-running-tasks
description: Orchestrate multi-phase background development using coding agents. Use when: (1) a project has multiple sequential tasks that should run autonomously without human intervention between steps, (2) the user wants continuous background work with crash recovery, stall detection, and progress reporting, (3) the user mentions "long-running tasks", "autonomous development", "background orchestration", or "multi-phase project". Not for: single one-shot tasks, interactive pairing, or work requiring human review between every step.
---

# Long-Running Task Orchestration

Run multi-phase projects autonomously using coding agents as workers and cron jobs as the orchestrator.

## The Problem

Coding agents are one-shot: they complete a task, exit, and nothing spawns the next one. Without orchestration, work stalls silently between tasks until a human notices.

## Architecture

```
Orchestrator (cron, every 10-30 min)
  │
  ├─ Lock file exists?  → another orchestrator is running, exit
  ├─ .pause file exists? → paused, skip spawning
  ├─ Worker PID alive?   → check for stall, report status
  └─ No worker?          → read TODO.md → spawn next task
                               │
                               ▼
                          Worker (coding agent session)
                            - Read project context + TODO.md
                            - Implement one task
                            - Run tests
                            - Commit + push
                            - Update TODO.md
                            - Write completion marker
                            - Exit
```

## Setup

### 1. Create TODO.md in the project root

Structured task queue. Each task must be self-contained enough for a cold-start agent:

```markdown
# TODO

## Phase 1 — Name (IN PROGRESS)
- [x] Completed task
- [ ] **Task title**
      What to do, which files to touch, acceptance criteria.
- [ ] Another task...

## Phase 2 — Name (QUEUED)
- [ ] Task...
```

Mark dependent-on-external tasks with `BLOCKED:` prefix so the orchestrator skips them.

### 2. Create a project context file

Cold-start context for agents. Include: stack, architecture, key commands, current phase, environment setup. Keep under 100 lines.

See `references/context-file-template.md` for a template.

### 3. Set up the orchestrator cron

Use the OpenClaw `cron` tool with `sessionTarget: "isolated"` and `payload.kind: "agentTurn"`.

See `references/orchestrator-cron.md` for the full cron configuration and prompt template.

### 4. Spawn the first worker manually

Start the first task yourself. The orchestrator takes over after this:

```bash
cd /path/to/project && nohup <agent-command> '<task prompt>' > /tmp/worker.log 2>&1 &
echo $! > /tmp/lrt-worker.pid
```

## Worker Rules

Every worker prompt must include these instructions. See `references/worker-prompt-template.md` for a copy-paste template.

1. **Read context first** — project context file + TODO.md
2. **One task only** — pick the first unchecked, non-BLOCKED item
3. **Test before commit** — run the test suite; fix failures before proceeding
4. **Update TODO.md** — check off the completed item
5. **Commit + push** — use the project's commit convention
6. **Write PID cleanup** — remove `/tmp/lrt-worker.pid` on exit
7. **Signal completion** — `openclaw system event --text "Done: <summary>" --mode now`
8. **Never exit silently** — if blocked, commit what you have with a note explaining why

## Orchestrator Logic

The orchestrator cron runs this sequence every interval:

```
1. Acquire lock (/tmp/lrt-orchestrator.lock) → if locked, exit immediately
2. Check /tmp/lrt-worker.pid:
   a. PID alive + recent commit (< 30 min) → report status, release lock, exit
   b. PID alive + stale commit (> 30 min)  → kill worker, remove PID file
   c. PID dead or no PID file              → continue to step 3
3. Check for .pause file in project root → if exists, report "paused", release lock, exit
4. Read TODO.md → find first unchecked non-BLOCKED task
   - All done? → report "all tasks complete", release lock, exit
   - Found task? → spawn worker, write PID to /tmp/lrt-worker.pid, verify with kill -0
5. Report what was spawned (task name, latest commit hash)
6. Release lock
```

## Failure Recovery

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Worker crashed | PID dead + no new commit | Respawn same task |
| Worker hung | PID alive + no commit in 30 min | Kill, respawn |
| Tests failing | Worker fixes inline | If stuck 2+ attempts, mark BLOCKED, move on |
| Context overflow | Agent exits mid-task | Respawn with narrower scope hint |
| Double-spawn | Lock file exists | Second orchestrator exits immediately |

## Pause and Resume

Create `.pause` in the project root to stop the orchestrator from spawning new workers:

```bash
touch /path/to/project/.pause    # pause
rm /path/to/project/.pause       # resume
```

The orchestrator still runs on schedule but skips spawning and reports "paused" status.

## Progress Reporting

Avoid repeating the same status every interval:

- Track the last-reported commit hash (store in `/tmp/lrt-last-commit`)
- Only send a substantive report when there's a NEW commit
- Include diff stats (`git diff --stat HEAD~1`), not a full inventory
- If nothing changed since last report, send one line: "no new commits since [hash]"

## Anti-Patterns

- **Reporter-only cron** — checks status but never spawns work
- **Monolithic prompts** — giving a worker 10 tasks at once; it'll do 2 and exit
- **No TODO.md** — orchestrator can't determine what's next without a structured task list
- **No PID tracking** — using `pgrep` pattern matching leads to false positives from unrelated processes
- **Polling in main session** — don't `process poll` in a loop; let the cron handle scheduling
