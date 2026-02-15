# bmad-yolo

Autonomous BMAD Phase 4 implementation for Claude Code.

## What It Does

`/yolo-story` runs the full BMAD implementation cycle in a loop:
Create Story -> Implement -> Adversarial Code Review (subagent) -> Fix -> Commit -> Next Story

It creates a git branch, implements every story in the epic, and pushes clean commits.
You start it, go to sleep, wake up to a completed epic.

`/yolo-revert` safely discards the yolo branch and returns to main.

## Requirements

- A BMAD project with `_bmad/` installed (via `npx bmad-method install`)
- `epics.md` and `sprint-status.yaml` in your project
- `architecture.md` and `prd.md` for project context
- Git initialized with a remote
- Phases 1-3 complete (Brief, PRD, Architecture, Epics)
- Sprint planning complete (sprint-status.yaml populated)

## Install

Copy the `bmad-yolo/` directory into your Claude Code plugins location, or:

```bash
# From your project root
cp -r bmad-yolo ~/.claude/plugins/bmad-yolo
```

## Usage

```
/yolo-story       # Start autonomous implementation
/yolo-revert      # Discard yolo branch, return to main
```

## How It Works

1. Reads `_bmad/bmm/config.yaml` to find project artifacts
2. Reads `sprint-status.yaml` to find the next story
3. Creates `yolo/{epic-name}` git branch
4. For each story:
   - **Create:** Follows `_bmad/bmm/workflows/4-implementation/create-story/` workflow
   - **Implement:** Follows `_bmad/bmm/workflows/4-implementation/dev-story/` workflow (TDD)
   - **Review:** Spawns `yolo-reviewer` subagent for independent adversarial code review
   - **Fix:** Addresses all findings (up to 3 rounds)
   - **Commit:** Atomic commit per story, push to remote
5. Loops until epic is complete

## Safety

- All work on a `yolo/{epic-name}` branch — main is never touched
- Code review in separate subagent context for honest, unbiased review
- Each story is an atomic commit — easy to cherry-pick or revert
- `/yolo-revert` nukes everything and returns to main
- Destructive operations (DB drops, deployments) are logged but never executed

## What It Won't Do

- Merge into main (that's your call after reviewing)
- Execute destructive database operations
- Deploy to any environment
- Complete OAuth or interactive flows
- Force push, rebase, or delete main

## Plugin Structure

```
bmad-yolo/
├── .claude-plugin/
│   └── plugin.json              # Plugin metadata
├── skills/
│   └── yolo-story/
│       ├── SKILL.md             # Main autonomous loop controller
│       ├── workflow-guide.md    # How to read & follow BMAD workflow files
│       ├── state-machine.md     # Sprint status transitions, stop conditions
│       └── compact-anchor.md    # Re-injection instructions after /compact
│   └── yolo-revert/
│       └── SKILL.md             # Branch revert skill
├── agents/
│   └── yolo-reviewer.md         # Adversarial code review subagent
└── README.md
```

## License

MIT
