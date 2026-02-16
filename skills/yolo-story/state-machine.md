# Sprint Status State Machine

## Sprint Status YAML Format

The actual `sprint-status.yaml` uses this structure:

```yaml
# generated: {date}
# project: {project_name}
# project_key: {project_key}
# tracking_system: file-system
# story_location: "{implementation_artifacts}"

execution_mode: yolo-story    # or yolo-story-hitl — added at startup

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

## HITL Section (yolo-story-hitl only)

When running `/yolo-story-hitl`, sprint-status.yaml includes:

```yaml
hitl:
  telegram_chat_id: "123456789"
  n8n_webhook_url: "https://your-n8n.instance/webhook/yolo-story"
  enabled: true
```

- Added during HITL startup sequence (before normal startup)
- `telegram_chat_id` is sensitive — sprint-status.yaml MUST be in `.gitignore`
- If `enabled: false` or section is missing, notifications are silently skipped
- The orchestrator reads this section at the start of each story to route notifications

## Story Status Transitions (STRICT)

```
backlog -> ready-for-dev -> in-progress -> review -> done
```

Never skip states. Never go backwards except on revert.

| Transition | Triggered By | Action |
|---|---|---|
| `backlog` -> `ready-for-dev` | yolo-story-creator (Opus) | Story file created with full context |
| `ready-for-dev` -> `in-progress` | yolo-developer (Sonnet) | Implementation started |
| `in-progress` -> `review` | yolo-developer completion | All tasks done, tests pass |
| `review` -> `done` | yolo-reviewer clean pass | Review passed, story complete |
| `review` -> `in-progress` | yolo-reviewer with fixes | Issues found, needs more work |

## Epic Status Transitions

| Transition | Triggered By |
|---|---|
| `backlog` -> `in-progress` | First story in epic moves to `ready-for-dev` |
| `in-progress` -> `done` | All stories in epic are `done` |

## YOLO-Specific Fields

```yaml
execution_mode: yolo-story    # or yolo-story-hitl — added at startup, removed on revert
```

Per-story tracking (added by orchestrator for richer state):
```yaml
# In the development_status section, stories may have additional yolo metadata
# tracked via comments above each story key:
# yolo-commit: abc1234
# yolo-flagged: true
```

## Atomic Operations

When updating sprint-status.yaml, follow this protocol:

1. **Read** the full YAML content
2. **Modify** only the specific field being updated
3. **Preserve** all comments, whitespace, and ordering
4. **Write** the complete file back
5. **Verify** by re-reading and checking the updated field

Never do partial writes. Never truncate the file. If the write fails, re-read and retry once.

## Subagent Tracking

The orchestrator tracks subagent IDs for session reuse:

| Agent | Tracking | Resume? |
|---|---|---|
| yolo-health-checker | Not tracked | No — stateless |
| yolo-story-creator | Not tracked | No — story file captures everything |
| yolo-developer | **agent_id saved** | **Yes** — resumed for fix cycles |
| yolo-validator | Not tracked | No — stateless |
| yolo-reviewer | Not tracked | No — fresh context intentional |

The developer's `agent_id` is valid only within a single story cycle. It resets between stories. If context is compacted and the agent_id is lost, spawn a fresh developer.

## Stop Conditions

1. All stories in active epic have status `done` -> STOP (epic complete)
2. Build failure after 3 retries AND no independent stories remaining -> STOP
3. No stories with status `backlog` or `ready-for-dev` -> STOP (sprint complete)

## Resume Logic

If sprint-status.yaml has `execution_mode: yolo-story` (or `yolo-story-hitl`) and stories are not all done:

| Current Status | Resume Action |
|---|---|
| `in-progress` | Spawn fresh yolo-developer from first unchecked task |
| `review` | Spawn fresh yolo-reviewer |
| `ready-for-dev` | Spawn yolo-developer |
| `backlog` | Spawn yolo-story-creator |

Finding the resume point:
1. Scan `development_status` top to bottom
2. Find the first story that is NOT `done`
3. Resume from that story's current status

## Story File Status Sync

The story file's `Status:` field and sprint-status.yaml MUST stay in sync:
- Always update sprint-status.yaml FIRST (it's the source of truth)
- Then update the story file's Status field
- If they diverge, sprint-status.yaml wins
