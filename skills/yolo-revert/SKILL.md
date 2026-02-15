---
name: yolo-revert
description: >
  Reverts a yolo-story branch back to main. Safely discards all changes made
  during autonomous implementation. Use when yolo-story produced bad results
  and you want to start over. Triggers on: "revert yolo", "undo yolo",
  "go back to main", "discard yolo branch".
disable-model-invocation: true
---

# yolo-revert

## Instructions

When invoked:

1. **Check current git status** — warn if there are uncommitted changes

2. **Show the user what will be lost:**
   - Current branch name
   - List all commits on the yolo branch that aren't on main:
     ```bash
     git log main..HEAD --oneline
     ```
   - Show total files changed:
     ```bash
     git diff main --stat
     ```

3. **Ask for ONE confirmation:**
   "This will discard all yolo changes and return to main. Confirm? (y/n)"

4. **On confirmation:**
   ```bash
   git checkout main
   git branch -D yolo/{branch-name}
   ```
   - Check if remote branch exists: `git ls-remote --heads origin yolo/{branch-name}`
   - If remote exists: `git push origin --delete yolo/{branch-name}`

5. **Clean up sprint-status.yaml:**
   - Read `_bmad/bmm/config.yaml` to find `implementation_artifacts` path
   - Read `{implementation_artifacts}/sprint-status.yaml`
   - Remove `execution_mode: yolo-story` field
   - Reset all `in-progress` and `review` story statuses back to `backlog`
   - Keep `done` stories as `done` (they were completed before this run or in a previous run)
   - Actually, reset ALL stories that were part of this yolo run back to `backlog`
     - To identify yolo stories: any story that changed status since yolo started
     - Simplest approach: reset everything that isn't `done` back to `backlog`
   - Save file, preserving all comments and structure

6. **Delete story files created during this yolo run:**
   - Story files in `{implementation_artifacts}/` that correspond to stories being reset
   - Only delete files that were created (not ones that existed before)
   - If unsure, leave them — the user can clean up manually

7. **Confirm completion:**
   ```
   Reverted to main. All yolo changes discarded.
   Branch yolo/{name} deleted (local + remote).
   Sprint status reset. Ready for a fresh run.
   ```

## Safety

- NEVER force-delete without showing what will be lost
- NEVER revert if already on main (or if no yolo branch exists)
- ALWAYS ask for confirmation — this is destructive
- NEVER delete the main branch
- If uncommitted changes exist, warn and ask whether to stash or discard
