---
name: yolo-story-hitl
description: >
  Autonomous BMAD Phase 4 with Telegram notifications. Same as yolo-story but sends
  detailed implementation changelogs to Telegram via n8n after each story step.
  You sleep. It codes. It tells you what happened.
  Triggers on: "yolo hitl", "yolo with notifications", "yolo telegram", "notify me".
disable-model-invocation: true
---

# yolo-story-hitl

Same autonomous loop as `/yolo-story` with Telegram notifications via n8n webhook after every story step. Non-blocking, fire-and-forget. You never pause for notifications.

## HITL STARTUP SEQUENCE

Before the normal yolo-story startup, configure notifications:

1. **Tell the user:** "Before we start, send any message to t.me/@yolostory_bot to activate the bot. Then give me your Telegram chat ID (numeric ID, not username)."

2. **Collect from user:**
   - `telegram_chat_id` — numeric Telegram chat ID
   - `n8n_webhook_url` — defaults to `https://auto.totalhumandesign.com/webhook/yolo-story` (only ask if user wants a custom URL)

3. **Write HITL config** to `{implementation_artifacts}/sprint-status.yaml`:
   ```yaml
   hitl:
     telegram_chat_id: "{provided_id}"
     n8n_webhook_url: "https://auto.totalhumandesign.com/webhook/yolo-story"
     enabled: true
   ```

4. **Gitignore check:** Ensure `sprint-status.yaml` is in the project's `.gitignore`. If not, add it. This file now contains a Telegram chat ID — it must NOT be committed.

5. **Send test notification:**
   ```bash
   curl -s -X POST "$N8N_WEBHOOK_URL" \
     -H "Content-Type: application/json" \
     -d '{"telegram_chat_id": "ID", "event": "test", "message": "YOLO HITL connected. Starting autonomous sprint."}'
   ```
   If curl fails, warn the user but continue — notifications are best-effort, never blocking.

6. **Proceed to normal startup sequence** (below).

## STARTUP SEQUENCE

Same as `/yolo-story`:

1. **Read `_bmad/bmm/config.yaml`** to resolve:
   - `planning_artifacts` path
   - `implementation_artifacts` path
   - `user_name`, `project_name`

2. **Read `{implementation_artifacts}/sprint-status.yaml`** to understand current state
   - Parse `development_status:` section
   - Check `hitl:` section — if `enabled: true`, notifications are active
   - Identify active epic and next story

3. **Read project context docs:**
   - `{planning_artifacts}/epics.md` (or `*epic*.md`)
   - `{planning_artifacts}/architecture.md`
   - `{planning_artifacts}/prd.md`
   - `**/project-context.md` if it exists

4. **Create/checkout git branch:** `yolo/{epic-name-slugified}`

5. **Add `execution_mode: yolo-story-hitl`** to sprint-status.yaml

6. **Begin the Per-Story Cycle**

## THE PER-STORY CYCLE (with Subagent Dispatch)

```
ORCHESTRATOR (Opus, main session — extended thinking)
│
├─ 1. Read sprint-status.yaml → identify next story
│     └─ Check hitl.enabled for notification routing
│
├─ 2. Dispatch yolo-health-checker (Haiku)
│     └─ Returns: HEALTHY or inconsistencies
│     └─ Auto-fixes: branch checkout, git stash
│     └─ If CRITICAL: stop and report
│
├─ 3. Dispatch yolo-story-creator (Opus) ← CRITICAL STEP
│     └─ Creates exhaustive story — zero ambiguity
│     └─ Update sprint-status: backlog → ready-for-dev
│     └─ [HITL] Send story_created notification
│
├─ 4. Dispatch yolo-developer (Sonnet) ← JUNIOR DEV
│     └─ SAVE agent_id for session reuse
│     └─ Follows story exactly, TDD, no decisions
│     └─ Update sprint-status: ready-for-dev → in-progress → review
│     └─ [HITL] Send story_implemented notification
│
├─ 5. Dispatch yolo-validator (Haiku) ← GATE
│     └─ Build pass? Tests pass? Tasks complete?
│     └─ If FAIL: resume yolo-developer → re-validate
│
├─ 6. Dispatch yolo-reviewer (Sonnet, FRESH context)
│     └─ Adversarial review with min 10 findings target
│     └─ Returns: BLOCKERS/HIGH/MEDIUM/LOW with file:line
│     └─ [HITL] Send review_findings notification
│
├─ 7. If issues: RESUME yolo-developer (saved agent_id)
│     └─ Fix ALL findings → re-validate → re-review
│     └─ Max 3 rounds → FLAG if unresolved
│     └─ [HITL] Send fix_cycle notification per round
│
├─ 8. Git commit + push (atomic, per story)
│     └─ Update sprint-status: review → done
│     └─ [HITL] Send story_done OR story_flagged notification
│
└─ 9. Next story (loop back to 1)
      └─ On epic complete: [HITL] Send epic_complete notification
```

### Step 1: HEALTH CHECK

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

Parse the structured output. If `HEALTH_CHECK: CRITICAL`, stop. Otherwise continue.

### Step 2: CREATE STORY (Opus)

Dispatch the story creator — this is the most important step:

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

Parse `STORY_CREATED: {path}` from output. Update sprint-status: `backlog` → `ready-for-dev`.

**[HITL] Send notification:** `story_created` — include story key, title, all tasks listed, all ACs listed, branch, progress.

### Step 3: IMPLEMENT (Sonnet)

Dispatch the developer:

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

**SAVE the returned agent_id** — this session will be resumed for fix cycles.

Parse `IMPLEMENTATION_COMPLETE` output for files, tests, deps.

**[HITL] Send notification:** `story_implemented` — include every file created/modified with descriptions, dependencies with versions, task count, test count, build status.

### Step 4: VALIDATE (Haiku Gate)

Dispatch the validator:

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

Parse findings by severity.

**[HITL] Send notification:** `review_findings` — include every finding with file:line, description, and severity. Include verdict and round number.

### Step 6: FIX CYCLE (Resume Sonnet)

If review found issues, **resume** the developer:

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

After fixes: re-validate (Step 4) → re-review (Step 5).

**Maximum 3 rounds.** Track finding counts:
- If findings DECREASE: progressing, continue
- If findings INCREASE: regression, FLAG immediately
- If 3 rounds exhausted: FLAG

**[HITL] Send notification:** `fix_cycle` — include every fix applied with severity, files modified, new deps, test count.

### Step 7: COMMIT & DONE

```bash
git add -A
git commit -m "feat(yolo): {story-key} - {title}

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
git push origin yolo/{epic-name}
```

Update sprint-status: `review` → `done`. Write completion brief in story file.

**[HITL] Send notification:** `story_done` — include commit hash, message, final file counts, test counts, review rounds, dependencies, key decisions, any human-check items. OR `story_flagged` — include all round summaries, unresolved blockers, human decisions needed.

### Step 8: NEXT STORY

Loop back to Step 1. When all stories are `done`:

**[HITL] Send notification:** `epic_complete` — include per-story summary (clean/flagged, round counts), totals (commits, files, tests, findings), branch, merge instructions, flagged story warnings.

## NOTIFICATION DISPATCH

### How to Send

Read `hitl` section from sprint-status.yaml. If `enabled: true`:

```bash
curl -s -X POST "{n8n_webhook_url}" \
  -H "Content-Type: application/json" \
  -d '{
    "telegram_chat_id": "{telegram_chat_id}",
    "event": "{event_type}",
    "message": "{full pre-formatted message}"
  }'
```

### Rules

- **Fire-and-forget.** Never wait for response. Never retry on failure. Never pause.
- **Build the full message in the orchestrator.** n8n just relays — no formatting logic there.
- **Messages must be full changelogs.** The human should understand exactly what happened without opening their laptop.
- If curl fails (network, n8n down), log a warning and continue. Notifications never block execution.
- If `hitl.enabled` is false or `hitl` section is missing, skip all notifications silently.

### Event Types and Message Templates

**`story_created`** — After Opus creates the story file:
```
STORY CREATED — {story_key}
"{story_title}"

Tasks ({count}):
  1. {task description}
  2. {task description}
  ...

Acceptance Criteria ({count}):
  {bullet list of all ACs}

Branch: yolo/{epic-name}
Progress: {done_count}/{total_count} stories — story created, ready for dev
```

**`story_implemented`** — After Sonnet completes all tasks:
```
STORY IMPLEMENTED — {story_key}

Files Created:
  + {path} — {description}
  ...

Files Modified:
  ~ {path} — {what changed}
  ...

Dependencies Added:
  {package}@{version}, ...
  {dev deps if any}

Tasks Completed: {done}/{total}
Tests: {written} written, {passing} passing
Build: {clean|fail}

Branch: yolo/{epic-name}
Progress: {done_count}/{total_count} stories — implementation done, going to review
```

**`review_findings`** — After each reviewer pass:
```
REVIEW FINDINGS — {story_key} (Round {n})

BLOCKERS ({count}):
  [{file}:{line}] {description}
  — {impact}
  ...

HIGH ({count}):
  [{file}:{line}] {description}
  — {impact}
  ...

MEDIUM ({count}):
  [{file}:{line}] {description}
  — {impact}
  ...

LOW ({count}):
  [{file}:{line}] {description}
  ...

Verdict: {PASS|FAIL} — {summary}
{Next action: going to fix cycle / clean pass}
```

**`fix_cycle`** — After each developer fix round:
```
FIX CYCLE {n} — {story_key}

Fixed ({fixed}/{total}):
  [{severity}] {description of fix}
  ...

Files Modified:
  ~ {path} — {what changed}
  + {path} — NEW: {description}
  ...

Dependencies Added:
  {if any}

Tests: {total} total ({new} new), all passing
Re-validating → sending to re-review...
```

**`story_done`** — After commit + status update:
```
YOLO STORY DONE — {story_key}

Commit: {short_hash}
Message: "feat(yolo): {story_key} — {title}"

Final State:
  Files created: {count}
  Files modified: {count}
  Tests: {count} (all passing)
  Review rounds: {count} ({findings} findings → {fixed} fixed)
  Dependencies added: {list}

Key Decisions:
  {bullet list of notable decisions}

Needs Human Check:
  {bullet list if any, or "None"}

Branch: yolo/{epic-name}
Progress: {done_count}/{total_count} stories done
```

**`story_flagged`** — If 3 rounds exhausted:
```
YOLO STORY FLAGGED — {story_key}

Could not resolve after 3 review rounds:

Round 1: {findings} findings ({blockers} blockers, {high} high, {medium} medium)
Round 2: {findings} findings — {improving|regressing}
Round 3: {findings} findings — {improving|REGRESSION}

Unresolved Blockers:
  [{file}:{line}] {description}
  — {what was attempted}
  ...

HUMAN DECISION NEEDED:
  1. {decision question}
  2. {decision question}

Committed as-is with FLAGGED marker.
Branch: yolo/{epic-name}
Progress: {done_count}/{total_count} stories done, {flagged_count} flagged
```

**`epic_complete`** — Final summary:
```
YOLO EPIC COMPLETE — {epic_name}

Stories: {total}/{total} done ({flagged_count} flagged)

{per-story list with status, round counts}

Totals:
  Commits: {count}
  Files created: {count}
  Files modified: {count}
  Tests written: {count} (all passing)
  Review findings: {total} total, {fixed} fixed, {unresolved} unresolved
  Dependencies added: {count} packages

Branch: yolo/{epic-name}
Ready for: git checkout main && git merge yolo/{epic-name}

{flagged story warnings if any}
```

## CONTEXT MANAGEMENT

Same as `/yolo-story`:

**Within a Story:** Full continuity. Developer agent_id preserved for resume.

**Between Stories:** Soft reset. Carry forward: completion briefs, project docs, codebase. Discard: implementation details, review findings, fix reasoning.

## COMPACT RECOVERY

If context has been compacted:
1. STOP and re-read this SKILL.md
2. Read sprint-status.yaml — source of truth
3. Check `execution_mode` — should be `yolo-story-hitl`
4. Check `hitl:` section — re-read chat_id and webhook_url
5. Resume from the current story's current step
6. If a developer agent_id was saved before compaction, it's lost — spawn fresh

## GIT OPERATIONS

Same as `/yolo-story`:
- **Creates branch:** `yolo/{epic-name}` at startup
- **Commits:** One atomic commit per story
- **Pushes:** After each commit
- **Gitignore:** sprint-status.yaml MUST be in .gitignore (contains chat_id)

Does NOT: merge, delete branches, force push, rebase.

## YOLO SCOPE

Same as `/yolo-story`. See `workflow-guide.md` for BMAD workflow paths and config resolution.

## STOP CONDITIONS

Same as `/yolo-story`:
1. All stories in active epic are `done`
2. Build failure after 3 attempts with no independent stories remaining
3. No more `backlog` or `ready-for-dev` stories

After stopping, send `epic_complete` notification and print the completion summary.

## TONE

Direct. No ceremony. Execute. Log decisions in commits and story files. Send notifications. Move fast.
