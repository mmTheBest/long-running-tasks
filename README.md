# Long-Running Task Orchestration

An [OpenClaw](https://github.com/openclaw/openclaw) skill for running multi-phase development projects autonomously with coding agents.

## The Problem

Coding agents like OpenAI Codex and Claude Code are one-shot: they finish a task, exit, and nothing spawns the next one. If your project has 15 phases, you're manually kicking off each step — often at 2am when the last one quietly finished.

## How It Works

Three pieces:

1. **TODO.md** — A structured task queue in your project root. Each task has status, description, and acceptance criteria.
2. **Orchestrator cron** — Runs every 10-15 minutes. Checks if a worker is alive; if not, reads TODO.md and spawns the next task.
3. **Worker rules** — Each agent session reads project context, implements one task, runs tests, commits, pushes, and updates TODO.md.

```
Orchestrator (cron, every 10-15 min)
  │
  ├─ Worker alive? → brief status report
  └─ Worker dead?  → read TODO → spawn next task
                         │
                         ▼
                    Worker (Codex / Claude Code)
                      - Read context
                      - Do one task
                      - Test → commit → push
                      - Update TODO.md
                      - Exit
```

## What's Included

| File | Purpose |
|------|---------|
| `SKILL.md` | Core guide — architecture, setup, worker rules, failure recovery, anti-patterns |
| `references/orchestrator-cron.md` | Cron config and prompt template |
| `references/claude-md-template.md` | Project context file template for cold-start agents |
| `references/worker-prompt-template.md` | Reusable task prompt with scoping tips |

## Handles

- **Crash recovery** — no process + no new commit → respawn same task
- **Hung workers** — process running >30min with no commit → kill and respawn
- **Blocked tasks** — prefix with `BLOCKED:`, orchestrator skips to the next independent task
- **Context overflow** — agent exits mid-task → orchestrator detects and respawns with smaller scope
- **Dead time** — short cron intervals + spawn verification to minimize idle gaps

## Codex vs Claude Code

- **Codex** — pure code changes where sandbox restrictions are fine
- **Claude Code** — tasks needing git push, API calls, or network access

## Setup

1. Install [OpenClaw](https://github.com/openclaw/openclaw)
2. Copy this skill to your OpenClaw skills directory
3. Create `TODO.md` and `CLAUDE.md` in your project root (see templates in `references/`)
4. Set up the orchestrator cron (see `references/orchestrator-cron.md`)
5. Spawn the first worker manually — the orchestrator takes over from there

## License

MIT
