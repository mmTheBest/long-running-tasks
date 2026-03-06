# Long-Running Tasks

**Autonomous multi-phase development with AI coding agents — without the silent stalls.**

[![ClawHub](https://img.shields.io/badge/ClawHub-long--running--tasks-blue)](https://clawhub.ai/skills/long-running-tasks)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## The Problem

AI coding agents (Claude Code, Codex, Cursor, etc.) are **one-shot**: they finish a task, exit, and nothing spawns the next one. Your project has 15 tasks? You're manually kicking off each step — often hours after the last one quietly finished at 3am.

Worse, agents doing real work (data processing, ML training, large builds) get **silently killed** by platform timeouts and never recover. You come back to find zero progress and no error message.

## Why Use This

| Without orchestration | With long-running-tasks |
|-----------------------|-------------------------|
| Agent finishes a task, exits — nothing starts the next one | Orchestrator reads TODO.md and spawns the next task automatically |
| Agent dies mid-task, nobody notices for hours | Multi-signal stall detection (commits + file activity + CPU) with automatic respawn |
| Platform timeouts silently kill data/ML work | System config guidance + per-workload timeout recommendations |
| Orchestrator falsely kills agents doing real work (downloads, training) | Configurable thresholds (30-120 min) + file activity checks prevent false kills |
| Long sessions overflow and degrade over time | Fresh cold-start worker per task — no context bloat |
| You keep checking "is it done yet?" | Progress reports delivered to Discord/Slack/Telegram via commit diffs |

## How It Works

```
Orchestrator (cron, every 10-30 min)
  │
  ├─ Worker alive + active?  → report status, exit
  ├─ Worker alive + stalled? → kill, respawn next task
  ├─ Worker dead?            → respawn next task
  └─ All tasks done?         → report complete
                                    │
                                    ▼
                               Worker (AI agent session)
                                 - Cold-start: read CLAUDE.md + TODO.md
                                 - Implement one task
                                 - Commit progress every 20-30 min
                                 - Run tests, push, signal completion
                                 - Exit cleanly
```

No polling loops. No manual intervention. Work continues autonomously until the task queue is empty or a blocker is hit.

## Features

- **Multi-signal stall detection** — checks commit age + file activity + process CPU, not just commits (prevents kill-loops on data-heavy tasks)
- **Configurable thresholds** — 30 min for code, 60-90 for data, 120 for ML training
- **Crash recovery** — dead worker + unchecked task → automatic respawn
- **Pause/resume** — `.pause` file stops spawning without disabling the cron
- **Cold-start workers** — fresh context per task, no session bloat
- **Intermediate commits** — workers commit every 20-30 min so progress is never lost
- **Multi-project support** — unique file slugs prevent collisions
- **Progress reporting** — commit-based diffs delivered to your channel
- **System config guidance** — documents the platform timeout fix most users miss

## Installation

This is an [OpenClaw](https://github.com/openclaw/openclaw) skill. Install via ClawHub:

```bash
clawhub install long-running-tasks
```

Or copy the skill files directly into your OpenClaw workspace.

### Prerequisites

Increase the OpenClaw embedded run timeout (default 600s is too short for real work):

```bash
openclaw config set agents.defaults.timeoutSeconds 1800  # 30 min
openclaw gateway restart
```

## Quick Start

1. Create `TODO.md` in your project root with a structured task queue
2. Create `CLAUDE.md` with project context + the progress protocol
3. Set up the orchestrator cron (see [orchestrator-cron.md](references/orchestrator-cron.md))
4. Spawn the first worker — the orchestrator handles everything after that

See [SKILL.md](SKILL.md) for the full setup guide.

## Use Cases

- **Feature development** — break features into tasks, let agents work overnight
- **Data pipelines** — ETL, preprocessing, model training across multiple stages
- **Refactoring** — systematic codebase changes across many files
- **Research experiments** — computational experiments with sequential dependencies
- **Test coverage** — generate tests module by module
- **Migrations** — database or API migrations with multiple steps

## Battle-Tested

Built for autonomous runs that last 24+ hours across 10+ sequential tasks — not toy demos. The multi-signal stall detection, kill-loop prevention, and system timeout guidance all came from real production failures where agents went silent for hours with no recovery. If your workflow involves data pipelines, long builds, or multi-step experiments that need to run overnight without hand-holding, this is what it's designed for.

## Requirements

- [OpenClaw](https://github.com/openclaw/openclaw) with cron support
- A coding agent CLI (Claude Code, Codex, or similar)
- Git repository with push access

## Documentation

- [SKILL.md](SKILL.md) — Full guide (architecture, setup, stall detection, worker rules, security)
- [references/orchestrator-cron.md](references/orchestrator-cron.md) — Cron config + prompt template with multi-signal detection
- [references/worker-prompt-template.md](references/worker-prompt-template.md) — Worker prompt template with intermediate commit rules
- [assets/context-file-template.md](assets/context-file-template.md) — Project context file template with progress protocol

## Keywords

AI agents, autonomous coding, background automation, Claude Code, Codex, cron orchestration, long-running tasks, multi-phase development, OpenClaw skill, task queue, unattended development, stall detection, crash recovery, silent stalls, kill loop prevention, data pipeline automation

## License

MIT
