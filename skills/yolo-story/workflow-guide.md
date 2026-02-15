# How to Follow BMAD Workflow Files

BMAD workflows live in the installed `_bmad/` directory. The skill cannot invoke
slash commands — it must load and follow the workflow files directly.

## Config Resolution

Before loading any workflow, read `_bmad/bmm/config.yaml` to resolve variables:

| Variable | Description |
|---|---|
| `planning_artifacts` | Directory for Phase 1-3 outputs (epics, architecture, PRD, UX) |
| `implementation_artifacts` | Directory for Phase 4 outputs (stories, sprint-status.yaml) |
| `user_name` | User's name for output messages |
| `communication_language` | Language for user communication |
| `document_output_language` | Language for generated documents |
| `user_skill_level` | Adjusts verbosity (irrelevant in YOLO mode) |

## Phase 4 Workflow Locations

These are the exact installed paths for each workflow:

### Create Story
```
_bmad/bmm/workflows/4-implementation/create-story/
├── workflow.yaml        # Variable definitions, input file patterns, config
├── instructions.xml     # Step-by-step execution logic (6 steps)
├── template.md          # Story file template (Story, AC, Tasks, Dev Notes, etc.)
└── checklist.md         # Quality competition prompt for validation
```

### Dev Story
```
_bmad/bmm/workflows/4-implementation/dev-story/
├── workflow.yaml        # Variable definitions, story_file, sprint_status
├── instructions.xml     # Step-by-step execution logic (10 steps, red-green-refactor)
└── checklist.md         # Enhanced Definition of Done checklist
```

### Code Review
```
_bmad/bmm/workflows/4-implementation/code-review/
├── workflow.yaml        # Variable definitions, input patterns for architecture/epics/UX
├── instructions.xml     # Adversarial review logic (5 steps, min 3-10 findings)
└── checklist.md         # Senior Developer Review validation checklist
```

### Sprint Planning (reference only — should already be done before yolo-story)
```
_bmad/bmm/workflows/4-implementation/sprint-planning/
├── workflow.yaml            # Variable definitions, tracking system config
├── instructions.md          # Sprint planning instructions
├── sprint-status-template.yaml  # Template for sprint-status.yaml
└── checklist.md             # Sprint planning checklist
```

### Adversarial Review Task
```
_bmad/core/tasks/review-adversarial-general.xml
```
Standalone adversarial review task. Expects content to review and optional `also_consider` areas. Requires minimum 10 findings. Used by the yolo-reviewer subagent.

## Executing a Workflow

1. **Read the workflow.yaml** first — resolve all `{variable}` references using config.yaml
2. **Read the instructions file** (`.xml` or `.md`) — this contains the step-by-step logic
3. **Follow instructions in order.** Each `<step>` must complete before the next
4. **For `<check>` blocks:** Evaluate the condition, execute the matching branch
5. **For `<action>` blocks:** Execute the action (read file, write file, run command, etc.)
6. **For `<ask>` blocks in YOLO mode:** Auto-continue. Pick the default/recommended option
7. **For HALT conditions in YOLO mode:** Attempt to resolve autonomously. Only truly halt on 3 consecutive failures
8. **Use the checklist** at the end of each workflow for validation

## Input File Discovery

Workflows use `input_file_patterns` in workflow.yaml to find documents:

```yaml
input_file_patterns:
  prd:
    whole: "{planning_artifacts}/*prd*.md"
    sharded: "{planning_artifacts}/*prd*/*.md"
    load_strategy: "SELECTIVE_LOAD"
  architecture:
    whole: "{planning_artifacts}/*architecture*.md"
    sharded: "{planning_artifacts}/*architecture*/*.md"
    load_strategy: "FULL_LOAD"
  epics:
    whole: "{planning_artifacts}/*epic*.md"
    sharded: "{planning_artifacts}/*epic*/*.md"
    load_strategy: "SELECTIVE_LOAD"
```

- `FULL_LOAD`: Read the entire document
- `SELECTIVE_LOAD`: Read only the sections relevant to the current story/epic
- Check for both whole files and sharded (split into folder) versions

## YOLO Mode Overrides

When executing workflows in YOLO mode:

| Normal Behavior | YOLO Override |
|---|---|
| `<ask>` prompts for user input | Auto-continue with default/recommended option |
| HALT on missing dependencies | Attempt to install/resolve autonomously |
| HALT on ambiguous requirements | Make best-judgment decision, log in story file |
| Menu choices (A/P/C/Y) | Always choose Continue |
| User confirmation prompts | Skip — auto-confirm |
| Web research step (create-story step 4) | Skip unless architecture explicitly requires it |

## Agent Mindset Per Workflow

- **Create Story:** Think like a Scrum Master. Systematic, thorough story definition. Extract EVERYTHING the dev needs from all artifacts.
- **Dev Story:** Think like a strict TDD developer. Tests first, implementation second. Never lie about tests. They must exist and pass. Execute continuously without pausing.
- **Code Review:** Delegated to yolo-reviewer subagent. Do NOT self-review. The separate context is the point.
