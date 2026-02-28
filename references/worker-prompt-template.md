# Worker Prompt Template

Use this as a starting point for task prompts given to coding agents.

## Template

```
You are working on [PROJECT_NAME]. Read CLAUDE.md and TODO.md for full context.

YOUR TASK:
[Copy the specific task description from TODO.md here]

STEPS:
1. Read CLAUDE.md for project context and commands
2. Read TODO.md for the full task list
3. Implement the task described above
4. Run tests: [TEST_COMMAND]
5. Fix any test failures before proceeding
6. Update TODO.md — check off the task you completed: change "- [ ]" to "- [x]"
7. Commit: git add -A && git commit -m 'update on "YYYY-MM-DD"'
8. Push: git push

When completely finished, run:
openclaw system event --text "Done: [BRIEF_SUMMARY]" --mode now

RULES:
- Do NOT skip tests. If tests fail, fix them.
- Do NOT exit without committing. If blocked, commit what you have with a note.
- Do NOT modify unrelated code outside the task scope.
```

## Scoping Tips

- **One task per worker.** Don't give 5 tasks hoping it'll do all of them.
- **Be specific about files.** "Add error handling to src/agent.py" > "Add error handling"
- **Include acceptance criteria.** "Tests must pass" or "Output must include X"
- **Set boundaries.** "Only modify files in src/eval/" prevents scope creep
