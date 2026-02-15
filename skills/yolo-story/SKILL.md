---
name: yolo-story
description: >
  Autonomous BMAD Phase 4 implementation loop. Creates stories, implements code,
  runs adversarial code review via subagent, fixes all issues, commits per story,
  and loops until the epic is complete. Use when you want to run an entire sprint
  autonomously. Triggers on: "yolo story", "run the sprint", "implement the epic",
  "start phase 4", "autonomous implementation", "yolo mode", "sleep mode".
disable-model-invocation: true
---

# yolo-story

Autonomous BMAD Phase 4 implementation. Execute the full story cycle in a loop until the epic is done.

## STARTUP SEQUENCE

When `/yolo-story` is invoked:

1. **Read `_bmad/bmm/config.yaml`** to resolve:
   - `planning_artifacts` path (where epics.md, architecture.md, prd.md live)
   - `implementation_artifacts` path (where sprint-status.yaml and story files live)
   - `user_name`, `project_name`

2. **Read `{implementation_artifacts}/sprint-status.yaml`** to understand current state
   - Parse the `development_status:` section completely
   - Identify the active epic (first `epic-N` with status `in-progress` or `backlog`)
   - Identify the next story (first `N-N-name` with status `backlog` or `ready-for-dev`)

3. **Read project context docs:**
   - `{planning_artifacts}/epics.md` (or `*epic*.md` pattern)
   - `{planning_artifacts}/architecture.md`
   - `{planning_artifacts}/prd.md`
   - `**/project-context.md` if it exists

4. **Create a git branch:** `yolo/{epic-name-slugified}` off the current branch
   - If branch already exists (resuming): `git checkout yolo/{epic-name}`
   - If new: `git checkout -b yolo/{epic-name}`

5. **Add `execution_mode: yolo-story` to sprint-status.yaml** as a top-level field so any future session knows this sprint is in autonomous mode

6. **Begin the Per-Story Cycle**

## THE PER-STORY CYCLE

For each story in the sprint:

### Step 1: CREATE STORY

Load and follow the actual Create Story workflow files:

- Read `_bmad/bmm/workflows/4-implementation/create-story/workflow.yaml` for variable resolution
- Read `_bmad/bmm/workflows/4-implementation/create-story/instructions.xml` and follow its steps
- Use `_bmad/bmm/workflows/4-implementation/create-story/template.md` as the story template
- Read `_bmad/bmm/workflows/4-implementation/create-story/checklist.md` for validation

The workflow does:
1. Find the first `backlog` story in sprint-status.yaml
2. Load epics file to extract story requirements, acceptance criteria, and context
3. Analyze architecture for technical guardrails
4. Load previous story learnings if story_num > 1
5. Generate the comprehensive story file with: story statement, acceptance criteria, tasks/subtasks, dev notes, project structure notes, references
6. Update sprint-status.yaml: story status -> `ready-for-dev`

**YOLO override:** Skip all user prompts and menu options. Auto-continue through every step. Skip web research (step 4 of create-story) unless architecture explicitly requires it.

### Step 2: IMPLEMENT STORY (Dev Story)

Load and follow the actual Dev Story workflow files:

- Read `_bmad/bmm/workflows/4-implementation/dev-story/workflow.yaml` for variable resolution
- Read `_bmad/bmm/workflows/4-implementation/dev-story/instructions.xml` and follow its steps
- Read `_bmad/bmm/workflows/4-implementation/dev-story/checklist.md` for definition-of-done

Update sprint-status.yaml: story status -> `in-progress`

The workflow does:
1. Find the `ready-for-dev` story and load it
2. Parse tasks/subtasks from the story file
3. For each task, follow red-green-refactor:
   - RED: Write failing tests first
   - GREEN: Implement minimal code to pass
   - REFACTOR: Clean up while tests stay green
4. Mark each task [x] when ALL validation gates pass
5. Run full test suite after each task to catch regressions
6. When all tasks done, update story Status -> `review`

Update sprint-status.yaml: story status -> `review`

**YOLO override:** Skip all user prompts. On HALT conditions (missing deps, config issues), attempt to resolve autonomously. Only truly HALT on 3 consecutive failures.

### Step 3: CODE REVIEW VIA SUBAGENT (Mandatory)

Spawn the `yolo-reviewer` subagent using the Task tool:

```
Task tool call:
  subagent_type: general-purpose
  prompt: [Load agents/yolo-reviewer.md content + story requirements + git diff + architecture context]
```

The subagent runs in a **separate context** — it does NOT see implementation reasoning, only the code.

Pass to the subagent:
- The story file (requirements, acceptance criteria, tasks)
- `git diff yolo/{epic-name}` for all changes in this story's commit range
- Architecture document for pattern compliance
- `_bmad/core/tasks/review-adversarial-general.xml` content for adversarial review protocol

Receive back: findings categorized as BLOCKERS, HIGH, MEDIUM, LOW with file:line references.

### Step 4: FIX ALL ISSUES (If Any Found)

Fix everything returned by the reviewer. All severities: blockers, high, medium, AND low. No issue gets skipped.

After fixes:
- Re-run all tests
- Re-run build

### Step 5: RE-REVIEW VIA SUBAGENT (Mandatory After Fixes)

Spawn `yolo-reviewer` again with the updated diff.

- If new issues found -> back to Step 4
- **Maximum 3 fix-review cycles.** If still failing after 3 rounds:
  - Add a `## FLAGGED` section in the story file explaining what couldn't be resolved
  - Set `flagged: true` in sprint-status.yaml for this story
  - Commit the story as-is with: `feat(yolo): [story-key] - [title] [FLAGGED]`
  - Continue to next story
- If clean -> proceed to Step 6

### Step 6: COMMIT & MARK DONE

```bash
git add -A
git commit -m "feat(yolo): [story-key] - [story title summary]

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
git push origin yolo/{epic-name}
```

Update sprint-status.yaml: story status -> `done`

Write a **Story Completion Brief** (max 15 lines) at the end of the story file:
- What was built
- Key files changed
- Key decisions made
- Anything the next story should know

**-> NEXT STORY (back to Step 1)**

## CONTEXT MANAGEMENT

**Within a Story:** Full continuity. Keep all context from create -> implement -> review -> fix -> done.

**Between Stories:** Soft reset after marking done:
- The Story Completion Brief is the carry-forward artifact
- For the NEXT story, rely on: completion briefs from previous stories, original project docs (epics, architecture, PRD), and the actual codebase
- Do NOT carry forward implementation details, review findings, or fix reasoning from previous stories

## COMPACT RECOVERY (CRITICAL)

If context has been compacted or you're unsure of your current state:
1. STOP and re-read this entire SKILL.md
2. Read sprint-status.yaml — it is your source of truth
3. Check `execution_mode` field — if it says `yolo-story`, you are in autonomous mode
4. Check which stories are `done`, which is `in-progress`, which is next
5. Read completion briefs from done stories for context
6. Resume the cycle from the current story's current step
Never guess your position. Always verify from sprint-status.yaml.

## GIT OPERATIONS

This skill DOES perform:
- **Creates branch:** `yolo/{epic-name}` at startup
- **Commits:** After each story completion (one atomic commit per story)
- **Pushes:** After each commit to the remote

This skill does NOT:
- Merge into main
- Delete branches
- Force push
- Rebase

The branch is the safety net. Everything stays on `yolo/{epic-name}` until the human reviews and merges.

## YOLO SCOPE — JUST DO IT

- Install dependencies (npm install, pip install, etc.)
- Run migrations if code-only (Prisma generate, TypeORM migrations, Drizzle push)
- Create/modify config files
- Run build commands
- Run test suites
- Modify any project files needed for implementation
- Create new files, directories, whatever the story needs

## YOLO SCOPE — STILL DEFER

Even in YOLO mode, these get logged as comments in the story file and NOT executed:
- Database operations that destroy production data (DROP, TRUNCATE on existing tables)
- Operations requiring external API keys/credentials that aren't present
- Operations requiring manual browser/device testing
- CI/CD pipeline triggers
- Deployment to any environment
- Anything requiring interactive user input (OAuth flows, interactive prompts)

## STOP CONDITIONS

The loop stops when:
1. **All stories in the active epic are `done`** — epic complete
2. **Build failure that cannot be resolved after 3 attempts** — flag it, try next story if independent, otherwise stop
3. **No more `backlog` or `ready-for-dev` stories in sprint-status.yaml** — sprint complete

After stopping, print:

```
═══ YOLO COMPLETE ═══
Epic: {name}
Stories completed: X/Y
Branch: yolo/{epic-name}
Flagged stories: {list any flagged}
Ready for: git checkout main && git merge yolo/{epic-name}
═══════════════════════
```

## TONE

Direct. No ceremony. No "I'll now proceed to..." narration. Execute. Log decisions in commit messages and story files. Move fast.
