---
name: long-running-tasks
description: Orchestrate multi-phase background development using coding agents (Codex, Claude Code). Use when a project requires continuous autonomous work across multiple tasks/phases — spawning workers, detecting stalls, recovering from failures, and reporting progress. Solves the "fire-and-forget" problem where one-shot agent sessions finish and nobody spawns the next task.
---

# Long-Running Task Orchestration

Run multi-phase projects autonomously using coding agents as workers and cron jobs as orchestrators.

## The Core Problem

Coding agents (Codex, Claude Code) are one-shot: they do one task, exit, and nothing spawns the next. Without orchestration, work stalls silently until a human notices.

## Architecture

```
┌─────────────────────────────────────────┐
│  Orchestrator Cron (every 15-30 min)    │
│  - Is a worker alive? (pgrep)           │
│  - If yes → brief status report         │
│  - If no  → read TODO → spawn next task │
│  - Report what was spawned              │
└────────────┬────────────────────────────┘
             │ spawns
             ▼
┌─────────────────────────────────────────┐
│  Worker (Codex / Claude Code session)   │
│  - Reads CLAUDE.md + TODO.md            │
│  - Implements one task                  │
│  - Runs tests                           │
│  - Commits + pushes                     │
│  - Checks off TODO item                 │
│  - Sends completion event               │
└─────────────────────────────────────────┘
```

## Setup Checklist

### 1. Create TODO.md in the project root

Each task must be self-contained enough for a cold-start agent to pick up:

```markdown
# Project TODO

## Phase N — Name (IN PROGRESS)
- [x] Completed task
- [ ] **Next task title**
      What to do, which files to touch, what tests to run.
      Expected output or acceptance criteria.
- [ ] Another task...

## Phase N+1 — Name (QUEUED)
- [ ] Task...

## Rules
- Commit message format: update on "YYYY-MM-DD"
- Always run tests before committing
- Push to origin after every commit
```

### 2. Create CLAUDE.md in the project root

Context file for cold-start agents. Include: stack, architecture, key commands, current phase. See `references/claude-md-template.md`.

### 3. Set up the orchestrator cron

Use the OpenClaw cron tool. See `references/orchestrator-cron.md` for the full prompt template.

Key settings:
- **Schedule:** `*/15 * * * *` or `*/30 * * * *` (every 15-30 min)
- **Model:** Use a cheap model (Sonnet) — orchestrator does light work
- **Timeout:** 180s (needs time to spawn a worker)
- **Session target:** `isolated` (don't pollute main session)

### 4. Spawn the first worker manually

Start the first task yourself, then the orchestrator takes over:

```bash
cd /path/to/project && codex --yolo exec '<task prompt>'
# or
cd /path/to/project && claude --dangerously-skip-permissions '<task prompt>'
```

## Worker Session Rules

Every worker prompt MUST include these instructions:

1. **Read context first:** `Read CLAUDE.md and TODO.md`
2. **Do one task:** Pick the first unchecked item from TODO.md
3. **Test before commit:** Run the test suite; fix failures before proceeding
4. **Update TODO:** Check off completed items in TODO.md
5. **Commit + push:** Use the project's commit convention
6. **Signal completion:** End with `openclaw system event --text "Done: <summary>" --mode now`
7. **Never exit silently:** If blocked, commit what you have with a note explaining why

## Failure Recovery

The orchestrator handles these automatically:

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Worker crashed | No process + no new commit | Respawn same task |
| Worker hung | Process age > 30min, no new commit | Kill + respawn |
| Tests failing | Worker should fix inline | If stuck, skip to next task with a note |
| API/network down | All queries fail | Worker writes error report, orchestrator retries later |
| Context overflow | Claude Code exits mid-task | Orchestrator detects no commit, respawns with smaller scope |

## Progress Reporting

Avoid the "parrot problem" (repeating the same status):

- Track last-reported commit hash
- Only report when there's a NEW commit
- Include diff stats, not full inventory
- If nothing changed, say "no new commits" in one line

## Continuous Learning (from ECC)

When a worker fails and recovers, or when a pattern emerges:

1. **Log the failure** in a `devlog.md` or `memory/` file
2. **Update TODO.md** with lessons (e.g., "Note: source ~/.zshrc before API calls")
3. **Update CLAUDE.md** if the failure reveals missing context for cold-start agents

This prevents future workers from hitting the same issue.

## Minimizing Dead Time

The biggest risk is the gap between a worker finishing and the orchestrator noticing. Mitigations:

1. **Short cron interval** — 10 minutes, not 30. The dead gap equals at most one interval.
2. **Mark BLOCKED tasks** — Prefix blocked items with `BLOCKED:` so the orchestrator skips them and moves to the next task instead of stalling.
3. **Verify spawn** — After spawning, run `sleep 2 && pgrep -f 'codex'` to confirm the worker actually started.
4. **Skip-not-block** — If a task depends on an external fix (API access, human input), mark it BLOCKED and let the orchestrator proceed to independent tasks.

## Anti-Patterns

- **Reporter-only cron:** A cron that checks status but never spawns work. Useless.
- **Monolithic prompts:** Giving a worker all 20 tasks at once. It'll do 3 and exit.
- **No TODO.md:** Orchestrator can't figure out what's next without a structured task list.
- **No completion signal:** Worker finishes silently, orchestrator waits until next tick to notice.
- **Polling loops in main session:** Don't `process poll` in a loop. Let the cron handle it.

## Choosing Between Codex and Claude Code

| Factor | Codex (`codex --yolo exec`) | Claude Code (`claude --dangerously-skip-permissions`) |
|--------|---------------------------|-----------------------------------------------------|
| Sandbox | Yes (network-restricted) | No (full host access) |
| Git push | Often blocked in sandbox | Works |
| ECC agents | No | Yes (`@planner`, `@python-reviewer`, etc.) |
| Best for | Pure code changes, tests | Tasks needing network, git, or multi-agent review |

**Recommendation:** Use Claude Code for tasks requiring git push or API calls. Use Codex for pure implementation tasks where sandbox is fine.

## Quick Reference

```
# Check if worker is alive
pgrep -f 'codex|claude' | head -5

# Check latest commit
cd /project && git log --oneline -1

# Spawn Codex worker
cd /project && nohup codex --yolo exec '<prompt>' > /tmp/worker.log 2>&1 &

# Spawn Claude Code worker
cd /project && nohup claude --dangerously-skip-permissions '<prompt>' > /tmp/worker.log 2>&1 &

# Kill stuck worker
pkill -f 'codex.*exec'
```
