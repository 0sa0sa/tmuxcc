# Agent Instructions

## Repository Setup

This is a forked repository.

- **origin**: `0sa0sa/tmuxcc` (this fork)
- **upstream**: `nyanko3141592/tmuxcc` (original)

## PR / Push Guidelines

**IMPORTANT: Do NOT push or create PRs to the upstream repository (`nyanko3141592/tmuxcc`).**

All changes should stay within this fork (`0sa0sa/tmuxcc`).

When creating PRs, always target this fork:

```bash
# Correct - PR within this fork
gh pr create --repo 0sa0sa/tmuxcc --base master --head <branch-name>

# WRONG - Do not do this
gh pr create --repo nyanko3141592/tmuxcc ...
```

## Workflow

1. Create feature branch from `master`
2. Make changes and commit
3. Push to `origin` (this fork)
4. Create PR targeting `0sa0sa/tmuxcc` master branch
