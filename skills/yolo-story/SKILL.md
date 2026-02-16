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

**Model tiering:** Opus thinks. Sonnet executes. Haiku validates.

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

4. **Gitignore check:** Ensure `sprint-status.yaml` is in the project's `.gitignore`. If not, add it before any commits.

5. **Create a git branch:** `yolo/{epic-name-slugified}` off the current branch
   - If branch already exists (resuming): `git checkout yolo/{epic-name}`
   - If new: `git checkout -b yolo/{epic-name}`

6. **Add `execution_mode: yolo-story` to sprint-status.yaml** as a top-level field so any future session knows this sprint is in autonomous mode

7. **Begin the Per-Story Cycle**

## THE PER-STORY CYCLE (Subagent Dispatch)

```
ORCHESTRATOR (Opus, main session — extended thinking)
│
├─ 1. Read sprint-status.yaml → identify next story
│
├─ 2. Dispatch yolo-health-checker (Haiku)
│     └─ Returns: HEALTHY or inconsistencies
│     └─ Auto-fixes: branch checkout, git stash
│
├─ 3. Dispatch yolo-story-creator (Opus) ← CRITICAL STEP
│     └─ Creates exhaustive story — zero ambiguity
│     └─ Update sprint-status: backlog → ready-for-dev
│
├─ 4. Dispatch yolo-developer (Sonnet) ← JUNIOR DEV
│     └─ SAVE agent_id for session reuse
│     └─ Follows story exactly, TDD, no decisions
│     └─ Update sprint-status: ready-for-dev → in-progress → review
│
├─ 5. Dispatch yolo-validator (Haiku) ← GATE
│     └─ Build pass? Tests pass? Tasks complete?
│     └─ If FAIL: resume yolo-developer → re-validate
│
├─ 6. Dispatch yolo-reviewer (Sonnet, FRESH context)
│     └─ Adversarial review with min 10 findings target
│     └─ Returns: BLOCKERS/HIGH/MEDIUM/LOW with file:line
│
├─ 7. If issues: RESUME yolo-developer (saved agent_id)
│     └─ Fix ALL findings → re-validate → re-review
│     └─ Max 3 rounds → FLAG if unresolved
│
├─ 8. Git commit + push (atomic, per story)
│     └─ Update sprint-status: review → done
│
└─ 9. Next story (loop back to 1)
```

### Step 1: HEALTH CHECK (Haiku)

Dispatch the health checker before each story:

```
Task tool:
  subagent_type: bmad-yolo:yolo-health-checker
  model: haiku
  prompt: |
    Check system health for yolo story execution.
    Epic name: {epic_name}
    Sprint status path: {implementation_artifacts}/sprint-status.yaml
    Implementation artifacts: {implementation_artifacts}
    [Include full yolo-health-checker.md instructions]
```

Parse the structured output. If `HEALTH_CHECK: CRITICAL`, stop and report. Otherwise continue.

### Step 2: CREATE STORY (Opus)

This is the most important step. Opus creates stories so exhaustive that Sonnet can't drift.

```
Task tool:
  subagent_type: bmad-yolo:yolo-story-creator
  model: opus
  prompt: |
    Create the next story for the sprint.
    Sprint status: {sprint-status.yaml content}
    Config: {config.yaml content}
    Planning artifacts path: {planning_artifacts}
    Implementation artifacts path: {implementation_artifacts}
    Previous story completion briefs: {if any}
    [Include full yolo-story-creator.md instructions]
```

Parse `STORY_CREATED: {path}` from output. Update sprint-status: `backlog` -> `ready-for-dev`. If first story in epic, set epic to `in-progress`.

### Step 3: IMPLEMENT (Sonnet)

Dispatch the junior developer:

```
Task tool:
  subagent_type: bmad-yolo:yolo-developer
  model: sonnet
  prompt: |
    Implement this story following BMAD dev-story workflow.
    Story file: {story_file_path}
    Story content: {story file content}
    Sprint status path: {implementation_artifacts}/sprint-status.yaml
    [Include full yolo-developer.md instructions]
```

**SAVE the returned agent_id** — this session will be resumed for fix cycles (saves ~50k tokens per fix round).

Parse `IMPLEMENTATION_COMPLETE` output for files created/modified, tests written, dependencies added.

### Step 4: VALIDATE (Haiku Gate)

Dispatch the validator before sending to review:

```
Task tool:
  subagent_type: bmad-yolo:yolo-validator
  model: haiku
  prompt: |
    Validate the implementation for story {story_key}.
    Story file: {story_file_path}
    [Include full yolo-validator.md instructions]
```

Parse `VALIDATION_RESULT: PASS|FAIL`.

If **FAIL**: Resume the developer (saved agent_id) to fix test/build failures, then re-validate. Do NOT proceed to review until validator passes.

### Step 5: REVIEW (Sonnet, Fresh Context)

Dispatch the reviewer — always fresh context (never resume):

```
Task tool:
  subagent_type: bmad-yolo:yolo-reviewer
  model: sonnet
  prompt: |
    [Load agents/yolo-reviewer.md content]
    Story file content: {story}
    Git diff: {git diff for this story}
    Architecture: {architecture.md content}
    Adversarial protocol: {review-adversarial-general.xml content}
```

Parse findings by severity from the structured output.

### Step 6: FIX CYCLE (Resume Sonnet)

If review found issues, **resume** the developer session:

```
Task tool:
  subagent_type: bmad-yolo:yolo-developer
  model: sonnet
  resume: {saved_agent_id}
  prompt: |
    Review findings to fix (ALL severities):
    {formatted findings with file:line}
    Fix every finding. Re-run tests after each fix.
```

After fixes: re-validate (Step 4) -> re-review (Step 5).

**Maximum 3 rounds.** Track finding counts:
- If findings DECREASE: progressing, continue
- If findings INCREASE: regression, FLAG immediately
- If 3 rounds exhausted: FLAG

### Step 7: COMMIT & DONE

```bash
git add -A
git commit -m "feat(yolo): {story-key} - {title}

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
git push origin yolo/{epic-name}
```

Update sprint-status: `review` -> `done`.

Write a **Story Completion Brief** at the end of the story file:
- What was built
- Key files changed
- Key decisions made
- Anything the next story should know

**-> NEXT STORY (back to Step 1)**

## FLAGGING A STORY

If 3 fix-review rounds exhausted or findings INCREASE:
1. Add `## FLAGGED` section in story file explaining unresolved issues
2. Set `flagged: true` comment above story key in sprint-status.yaml
3. Commit with: `feat(yolo): {story-key} - {title} [FLAGGED]`
4. Continue to next story

## CONTEXT MANAGEMENT

**Within a Story:** Full continuity. Developer agent_id preserved for resume.

**Between Stories:** Soft reset after marking done:
- Carry forward: completion briefs from previous stories, original project docs, the codebase
- Discard: implementation details, review findings, fix reasoning from previous stories

## COMPACT RECOVERY (CRITICAL)

If context has been compacted or you're unsure of your current state:
1. STOP and re-read this entire SKILL.md
2. Read sprint-status.yaml — it is your source of truth
3. Check `execution_mode` field — if it says `yolo-story`, you are in autonomous mode
4. Check which stories are `done`, which is `in-progress`, which is next
5. Read completion briefs from done stories for context
6. Resume the cycle from the current story's current step
7. If a developer agent_id was saved before compaction, it's lost — spawn fresh

Never guess your position. Always verify from sprint-status.yaml.

## GIT OPERATIONS

This skill DOES perform:
- **Creates branch:** `yolo/{epic-name}` at startup
- **Commits:** After each story completion (one atomic commit per story)
- **Pushes:** After each commit to the remote
- **Gitignore:** Ensures sprint-status.yaml is in .gitignore

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
YOLO COMPLETE
Epic: {name}
Stories completed: X/Y
Flagged stories: {list any flagged}
Branch: yolo/{epic-name}
Ready for: git checkout main && git merge yolo/{epic-name}
```

## TONE

Direct. No ceremony. No "I'll now proceed to..." narration. Execute. Log decisions in commit messages and story files. Move fast.
