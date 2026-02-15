# Sprint Status State Machine

## Sprint Status YAML Format

The actual `sprint-status.yaml` uses this structure:

```yaml
# generated: {date}
# project: {project_name}
# project_key: {project_key}
# tracking_system: file-system
# story_location: "{implementation_artifacts}"

execution_mode: yolo-story    # Added by yolo-story at startup

development_status:
  epic-1: in-progress
  1-1-user-authentication: done
  1-2-account-management: in-progress
  1-3-plant-data-model: backlog
  1-4-add-plant-manual: backlog
  epic-1-retrospective: optional

  epic-2: backlog
  2-1-personality-system: backlog
  2-2-chat-interface: backlog
  2-3-llm-integration: backlog
  epic-2-retrospective: optional
```

### Key format rules:
- Epic keys: `epic-N` with status: `backlog | in-progress | done`
- Story keys: `N-N-name` with status: `backlog | ready-for-dev | in-progress | review | done`
- Retrospective keys: `epic-N-retrospective` with status: `optional | done`
- Comments at the top (`# ...`) MUST be preserved when writing
- `execution_mode` is a top-level field added by yolo-story, not inside `development_status`

## Story Status Transitions (STRICT)

```
backlog -> ready-for-dev -> in-progress -> review -> done
```

Never skip states. Never go backwards except on revert.

| Transition | Triggered By | Action |
|---|---|---|
| `backlog` -> `ready-for-dev` | create-story workflow | Story file created with full context |
| `ready-for-dev` -> `in-progress` | dev-story workflow | Implementation started |
| `in-progress` -> `review` | dev-story completion | All tasks done, tests pass |
| `review` -> `done` | code-review clean pass | Review passed, story complete |
| `review` -> `in-progress` | code-review with fixes | Issues found, needs more work |

## Epic Status Transitions

| Transition | Triggered By |
|---|---|
| `backlog` -> `in-progress` | First story in epic moves to `ready-for-dev` |
| `in-progress` -> `done` | All stories in epic are `done` |

## YOLO-Specific Fields

```yaml
execution_mode: yolo-story    # Top-level, added at startup, removed on revert
```

Per-story tracking (optional, added by yolo-story for richer state):
```yaml
# In the development_status section, stories may have additional yolo metadata
# tracked via comments above each story key:
# yolo-commit: abc1234
# yolo-flagged: true
```

## Stop Conditions

1. All stories in active epic have status `done` -> STOP (epic complete)
2. Build failure after 3 retries AND no independent stories remaining -> STOP
3. No stories with status `backlog` or `ready-for-dev` -> STOP (sprint complete)

## Resume Logic

If sprint-status.yaml has `execution_mode: yolo-story` and stories are not all done:

| Current Status | Resume Action |
|---|---|
| `in-progress` | Resume dev-story implementation from first unchecked task |
| `review` | Resume code review cycle |
| `ready-for-dev` | Start dev-story implementation |
| `backlog` | Start create-story |

Finding the resume point:
1. Scan `development_status` top to bottom
2. Find the first story that is NOT `done`
3. Resume from that story's current status

## Story File Status Sync

The story file's `Status:` field and sprint-status.yaml MUST stay in sync:
- Always update sprint-status.yaml FIRST (it's the source of truth)
- Then update the story file's Status field
- If they diverge, sprint-status.yaml wins
