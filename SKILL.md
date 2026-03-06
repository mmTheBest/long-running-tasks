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
  ├─ Stale lock?         → clean up, continue
  ├─ Live lock?          → another orchestrator running, exit
  ├─ .pause file?        → skip spawning, report paused
  ├─ Worker PID alive?   → check for stall (multi-signal), report status
  └─ No worker?          → read TODO.md → spawn next task
                               │
                               ▼
                          Worker (coding agent session)
                            - Read project context + TODO.md
                            - Implement one task
                            - Commit intermediate progress every 20-30 min
                            - Commit + push final result
                            - Update TODO.md
                            - Exit
```

## Prerequisites: System Configuration

Before using this skill, ensure the OpenClaw embedded run timeout is sufficient for your tasks:

```bash
# Check current timeout (default is 600s / 10 min — too short for most real work)
openclaw config get agents.defaults.timeoutSeconds

# Set to 30 minutes (recommended minimum for data/ML/build tasks)
openclaw config set agents.defaults.timeoutSeconds 1800

# Restart gateway to apply
openclaw gateway restart
```

**Why this matters:** OpenClaw's embedded run timeout limits how long a single agent turn can execute. The default 600s is designed for conversational Q&A. Autonomous workers doing downloads, builds, data processing, or model training routinely need 15-60+ minutes per turn. If the timeout is too short, the agent gets killed mid-work, fails over through model profiles, and eventually goes permanently silent with no notification. This is the #1 cause of "silent stalls" in long-running tasks.

**Recommended values:**
| Workload | Timeout |
|----------|---------|
| Pure code changes, tests | 600s (default) |
| Builds, installs, API calls | 1200s (20 min) |
| Data processing, ML training, large downloads | 1800s (30 min) |
| Heavy compute (multi-hour transforms) | 3600s (60 min) |

## File Convention

All runtime files use a project slug to avoid collisions when orchestrating multiple projects:

```
/tmp/lrt-<project>-worker.pid         # worker PID
/tmp/lrt-<project>-orchestrator.lock  # orchestrator lock (contains orchestrator PID)
/tmp/lrt-<project>-last-commit        # last reported commit hash
/tmp/lrt-<project>-worker.log         # worker stdout/stderr
```

Choose a short, unique slug per project (e.g., `myapp`, `ra`, `blog`).

## Setup

### 1. Create TODO.md in the project root

Structured task queue. Each task must be self-contained enough for a cold-start agent:

```markdown
# TODO

## Phase 1 — Name (IN PROGRESS)
- [x] Completed task
- [ ] **Task title**
      What to do, which files to touch, acceptance criteria.
- [ ] BLOCKED: Task waiting on external input

## Phase 2 — Name (QUEUED)
- [ ] Task...
```

Tasks prefixed with `BLOCKED:` are skipped by the orchestrator.

### 2. Create a project context file

Cold-start context for agents (commonly named `CLAUDE.md` or `AGENTS.md`). Include: stack, architecture, runnable commands, current phase, environment setup. Keep under 100 lines.

**Must include these lines** (prevents the kill-loop anti-pattern):
```
IMPORTANT: Commit intermediate progress every 20-30 minutes.
Do NOT wait until the entire task is done to commit.
The orchestrator monitors commit timestamps to detect stalls.
```

See `assets/context-file-template.md` for a starter template to copy into your project.

### 3. Set up the orchestrator cron

Use the OpenClaw `cron` tool with `sessionTarget: "isolated"` and `payload.kind: "agentTurn"`.

Read `references/orchestrator-cron.md` for the full cron configuration, prompt template, and lock/stall logic.

### 4. Spawn the first worker manually

Start the first task yourself in safe (default) mode. The orchestrator takes over after this:

```bash
cd /path/to/project && nohup <agent-command> '<task prompt>' > /tmp/lrt-<project>-worker.log 2>&1 &
echo $! > /tmp/lrt-<project>-worker.pid
```

Use the agent's default permission mode for initial runs. See `references/orchestrator-cron.md` for agent command examples and security guidance.

## Worker Rules

Every worker prompt must include these instructions. See `references/worker-prompt-template.md` for a copy-paste template.

1. **Read context first** — project context file + TODO.md
2. **One task only** — pick the first unchecked, non-BLOCKED item
3. **Commit early and often** — commit intermediate progress every 20-30 minutes, not just at the end (prevents false stall detection)
4. **Test before final commit** — run the test suite; fix failures before proceeding
5. **Update TODO.md** — check off the completed item
6. **Commit + push** — use the project's commit convention
7. **Signal completion** — `openclaw system event --text "Done: <summary>" --mode now`
8. **Never exit silently** — if blocked, commit what you have with a note explaining why

Note: worker self-cleanup of the PID file is best-effort. The orchestrator is the real safety net — it checks whether the PID is still alive regardless of whether the file was cleaned up.

## Stall Detection (Multi-Signal)

The orchestrator must use **multiple signals** to determine if a worker is stalled. Commit age alone causes false positives — workers doing downloads, data processing, or model training may legitimately go 30+ minutes without committing.

**Signals to check (in order):**
1. **Commit age** — `git log -1 --format=%ct HEAD` vs current time
2. **File activity** — `find <project> -newer <last-commit-file> -type f -not -path '*/.git/*' -not -path '*/.venv/*' 2>/dev/null | wc -l` (new output files = active work)
3. **Process activity** — `ps -o rss,cputime -p <PID>` (growing RSS or CPU time = doing work)
4. **Log activity** — `stat -f %m /tmp/lrt-<project>-worker.log` (recent log writes)

**Stall thresholds:**

| Task type | Threshold | Rationale |
|-----------|-----------|-----------|
| Code changes, tests | 30 min | Fast cycle, should commit often |
| Data processing, downloads | 60-90 min | I/O-heavy, legitimate long waits |
| ML training, heavy compute | 120 min | May run hours between checkpoints |

**Default: 60 minutes.** Only declare a stall if ALL signals are inactive beyond the threshold. The old 30-minute commit-only check caused a kill-loop where workers were repeatedly spawned and killed for data-heavy tasks.

## Pause and Resume

```bash
touch /path/to/project/.pause    # pause — orchestrator skips spawning
rm /path/to/project/.pause       # resume
```

The orchestrator still runs on schedule but reports "paused" instead of spawning.

## Progress Reporting

- Track the last-reported commit hash in `/tmp/lrt-<project>-last-commit`
- Only send a substantive report when there's a NEW commit
- Include diff stats (`git diff --stat HEAD~1`), not a full inventory
- If nothing changed, one line: "no new commits since [hash]"

## Security

- **Sandbox first.** Run the orchestrator + worker pipeline in a test repo before pointing it at real projects. Validate that workers do what you expect.
- **Credential scoping.** Workers that commit and push need git credentials. Use a dedicated deploy key or machine account with minimum write access to the target repo. Never use your personal token with broad org access.
- **No secrets in context files.** Project context files (CLAUDE.md, TODO.md) must not contain API keys, tokens, or passwords. Reference where secrets are stored (e.g., "API key in ~/.zshrc") — never inline them.
- **Permission-bypass flags.** Agent CLIs often offer flags that skip safety prompts. Do not use these until you've verified the pipeline in safe mode. See `references/orchestrator-cron.md` for details.
- **Review before auto-push.** Consider disabling automatic `git push` in worker prompts during initial runs. Let workers commit locally; review the commits manually, then push. Enable auto-push only after you trust the output.
- **PID kill safety.** The orchestrator may kill a stalled worker by PID. Use unique project slugs (see File Convention) to avoid collisions with unrelated processes.

## Anti-Patterns

- **Reporter-only cron** — checks status but never spawns work
- **Monolithic prompts** — giving a worker 10 tasks; it'll do 2 and exit
- **No TODO.md** — orchestrator can't determine what's next without a structured task list
- **No PID tracking** — `pgrep` pattern matching hits false positives from unrelated processes
- **Shared `/tmp` paths** — running two projects without unique slugs causes PID/lock collisions
- **Polling in main session** — don't `process poll` in a loop; let the cron handle scheduling
- **Commit-only stall detection** — checking only commit age causes kill-loops on data-heavy tasks; always combine with file activity and process checks
- **Short stall thresholds** — 30 minutes is too aggressive for anything beyond pure code edits; use 60+ for data/ML work
- **No intermediate commits** — workers that only commit at the end look stalled for the entire duration; context files must instruct workers to commit every 20-30 min
- **Low embedded run timeout** — the default OpenClaw timeout (600s) kills workers mid-execution on data tasks; increase `agents.defaults.timeoutSeconds` before starting
