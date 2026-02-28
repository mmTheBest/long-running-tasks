# CLAUDE.md Template

Create this in the project root so cold-start agents have context.

```markdown
# Project Name

## Stack
- Language, framework, major dependencies
- Test framework and how to run tests
- Build system

## Architecture
- src/module/ — what it does
- src/other/ — what it does
- tests/ — test organization

## Commands
\`\`\`bash
# Run unit tests
command here

# Run all tests (including integration)
command here

# Lint / type check
command here
\`\`\`

## Commit Convention
- What format to use for commit messages

## Current Phase
What's being worked on now. What's next.

## Environment
- Required env vars and where they're set
- Any setup steps (venv, API keys, etc.)
\`\`\`
```

## Guidelines

- Keep under 100 lines — agents read this on every cold start
- Include runnable commands, not just descriptions
- Update when phases change or architecture shifts
- Don't include secrets — reference where they're stored (e.g., "API key in ~/.zshrc")
