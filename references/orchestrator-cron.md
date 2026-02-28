# Orchestrator Cron Setup

## Cron Job Configuration

```json
{
  "name": "Project orchestrator",
  "schedule": { "kind": "cron", "expr": "*/30 * * * *", "tz": "YOUR_TIMEZONE" },
  "sessionTarget": "isolated",
  "payload": {
    "kind": "agentTurn",
    "model": "anthropic/claude-sonnet-4-5",
    "timeoutSeconds": 180,
    "message": "SEE PROMPT TEMPLATE BELOW"
  },
  "delivery": {
    "mode": "announce",
    "channel": "discord",
    "to": "channel:YOUR_CHANNEL_ID"
  }
}
```

## Orchestrator Prompt Template

Customize the paths, channel ID, and user tag:

```
You are a development orchestrator for [PROJECT_NAME].

Step 1 — Check for running worker:
  Run: pgrep -f 'codex|claude' | head -5
  If a worker is running:
    - Run: cd [PROJECT_PATH] && git log --oneline -1
    - Send a one-line status to Discord channel [CHANNEL_ID] tagging <@USER_ID>
    - STOP here.

Step 2 — No worker running. Check what's next:
  Run: cd [PROJECT_PATH] && git log --oneline -1 && cat TODO.md

Step 3 — Find the first unchecked task (line starting with "- [ ]") and spawn a worker:
  Run: cd [PROJECT_PATH] && nohup codex --yolo exec '<TASK_PROMPT>' > /tmp/orchestrator-worker.log 2>&1 &

  The task prompt must include:
  - "Read CLAUDE.md and TODO.md for context"
  - The specific task to do (copy from TODO.md)
  - "Run tests before committing"
  - "Check off completed items in TODO.md"
  - "Commit with message: update on \"YYYY-MM-DD\""
  - "Push to origin"
  - "When finished: openclaw system event --text \"Done: <summary>\" --mode now"

Step 4 — Report to Discord channel [CHANNEL_ID] tagging <@USER_ID>:
  - Latest commit
  - What task you just spawned
  - Under 80 words

CRITICAL: Always spawn work if nothing is running and tasks remain. Never just report.
If all tasks in TODO.md are checked, report "All tasks complete" and stop.
```

## Adapting for Claude Code Workers

Replace the spawn command in Step 3:

```
cd [PROJECT_PATH] && nohup claude --dangerously-skip-permissions '<TASK_PROMPT>' > /tmp/orchestrator-worker.log 2>&1 &
```

Use Claude Code when:
- Tasks need network access (API calls, git push)
- You want ECC agents (@planner, @python-reviewer, etc.)
- The Codex sandbox blocks required operations

## Stall Detection Enhancement

Add to Step 1 to detect hung workers:

```
If a worker IS running but the latest commit is older than 30 minutes:
  - Run: pkill -f 'codex.*exec' || pkill -f 'claude.*dangerously'
  - Treat as "no worker running" and proceed to Step 2
  - Include "killed hung worker" in the Discord report
```

## Shutdown Condition

The orchestrator should stop spawning when:
- All items in TODO.md are checked (`- [x]`)
- Or a specific "DONE" marker exists in TODO.md
- At that point, send a final summary and recommend disabling the cron
