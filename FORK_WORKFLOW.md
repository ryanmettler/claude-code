# Fork Workflow Documentation

## Overview
This repository is a fork of `anthropics/claude-code`. We use a feature branch workflow to maintain custom changes while staying synchronized with upstream updates.

## Branch Structure
- `main`: Clean branch that mirrors upstream, used for syncing
- `dev`: Feature branch where all custom development happens

## Initial Setup (Already Done)
```bash
# Add upstream remote
git remote add upstream https://github.com/anthropics/claude-code.git

# Create and switch to dev branch
git checkout -b dev
git push -u origin dev
```

## Daily Workflow

### Working on Features
Always work on the `dev` branch:
```bash
git checkout dev
# Make your changes
git add .
git commit -m "Your commit message"
git push origin dev
```

### Syncing with Upstream (Recommended Weekly)
1. **Update main branch:**
   ```bash
   git checkout main
   git pull upstream main
   git push origin main
   ```

2. **Update dev branch with latest changes:**
   ```bash
   git checkout dev
   git rebase main
   # If there are conflicts, resolve them and continue:
   # git add .
   # git rebase --continue
   git push origin dev --force-with-lease
   ```

### Alternative: Merge instead of Rebase
If you prefer merging over rebasing:
```bash
git checkout dev
git merge main
git push origin dev
```

## Handling Conflicts
When rebasing/merging, conflicts may occur:
1. Edit conflicted files to resolve issues
2. Stage resolved files: `git add <file>`
3. Continue the operation: `git rebase --continue` or complete the merge
4. Push changes: `git push origin dev --force-with-lease` (for rebase)

## Best Practices
- Keep commits on `dev` focused and atomic
- Use descriptive commit messages
- Regularly sync with upstream to minimize conflicts
- Never force push to `main`
- Use `--force-with-lease` instead of `--force` when needed

## Current Status
- **Active Branch:** `dev` (your working branch)
- **Upstream:** `https://github.com/anthropics/claude-code.git`
- **Origin:** `https://github.com/ryanmettler/claude-code.git`