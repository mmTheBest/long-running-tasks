# Long-Running Tasks

**Autonomous multi-phase development with AI coding agents.**

[![ClawHub](https://img.shields.io/badge/ClawHub-long--running--tasks-blue)](https://clawhub.ai/skills/long-running-tasks)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

## The Problem

AI coding agents (Claude Code, Codex, Cursor, etc.) are **one-shot**: they complete a task, exit, and nothing spawns the next one. If your project has 15 tasks across 3 phases, you're manually kicking off each step — often hours after the last one quietly finished.

**Long-running tasks** solves this with a simple pattern: a cron-based orchestrator that detects when a worker finishes and automatically spawns the next task.

## How It Works

```
Orchestrator (cron, every 10-30 min)
  │
  ├─ Worker alive?  → report status
  └─ Worker done?   → read TODO.md → spawn next task
                          │
                          ▼
                     Worker (AI agent session)
                       - Read project context
                       - Implement one task
                       - Run tests
                       - Commit + push
                       - Exit
```

No polling loops. No manual intervention. Work continues autonomously until the task queue is empty.

## Features

- **Crash recovery** — dead worker + no commit → respawn same task
- **Stall detection** — worker alive but no progress for 30 min → kill and respawn
- **Pause/resume** — drop a `.pause` file to stop spawning without disabling the cron
- **Multi-project support** — unique file slugs prevent collisions
- **Progress reporting** — commit-based diffs, no parrot status updates
- **Security guidance** — credential scoping, sandbox-first, no secrets in context

## Installation

This is an [OpenClaw](https://github.com/openclaw/openclaw) skill. Install via ClawHub:

```bash
clawhub install long-running-tasks
```

Or copy the skill files directly into your OpenClaw workspace.

## Quick Start

1. Create `TODO.md` in your project root with a structured task queue
2. Create a project context file (`CLAUDE.md`) for cold-start agents
3. Set up the orchestrator cron using OpenClaw's cron tool
4. Spawn the first worker manually — the orchestrator takes over

See [SKILL.md](SKILL.md) for the full setup guide, worker rules, and security recommendations.

## Use Cases

- **Feature development** — break a feature into tasks, let agents work through the night
- **Refactoring** — systematic codebase changes across many files
- **Test coverage** — generate tests module by module
- **Documentation** — auto-generate docs for each component
- **Migrations** — database or API migrations with multiple steps

## Requirements

- [OpenClaw](https://github.com/openclaw/openclaw) with cron support
- A coding agent CLI (Claude Code, Codex, or similar)
- Git repository with push access

## Documentation

- [SKILL.md](SKILL.md) — Full guide (architecture, setup, worker rules, security)
- [references/orchestrator-cron.md](references/orchestrator-cron.md) — Cron configuration and prompt template
- [references/worker-prompt-template.md](references/worker-prompt-template.md) — Worker prompt template
- [assets/context-file-template.md](assets/context-file-template.md) — Project context file template

## Keywords

AI agents, autonomous coding, background automation, Claude Code, Codex, cron orchestration, long-running tasks, multi-phase development, OpenClaw skill, task queue, unattended development

## License

MIT
