# Compact Recovery Protocol

This file exists to survive /compact events.

When Claude's context is compacted during a yolo-story session:

1. The conversation summary will mention yolo-story or autonomous implementation
2. sprint-status.yaml contains `execution_mode: yolo-story` — this is the persistent flag
3. On detecting either signal, Claude must:
   a. Re-read the full yolo-story SKILL.md
   b. Re-read sprint-status.yaml for current position
   c. Re-read completion briefs from any done stories
   d. Resume exactly where it left off

This instruction is also embedded directly in SKILL.md (under COMPACT RECOVERY)
so it survives even if this supporting file isn't loaded after compact.

## Detection Signals

Any of these indicate yolo-story was active:
- `execution_mode: yolo-story` in sprint-status.yaml
- A `yolo/*` git branch exists and is checked out
- Conversation summary mentions "yolo", "autonomous implementation", or "sleep mode"
- Story files show recent modifications with yolo commit patterns

## Recovery Steps (Exact)

```
1. Read this SKILL.md completely
2. Read _bmad/bmm/config.yaml for path resolution
3. Read {implementation_artifacts}/sprint-status.yaml
4. Parse development_status — find first non-done story
5. Read that story file to determine current step:
   - Status: backlog -> needs create-story
   - Status: ready-for-dev -> needs dev-story
   - Status: in-progress -> needs to resume dev-story (check unchecked tasks)
   - Status: review -> needs code review
6. Read completion briefs from all done stories (last 15 lines of each)
7. Resume the cycle
```

## What NOT to Do After Compact

- Do NOT start over from the beginning
- Do NOT re-create stories that are already done
- Do NOT re-implement completed tasks
- Do NOT ask the user what to do — check sprint-status.yaml and continue
