# bmad-yolo

Autonomous BMAD Phase 4 implementation for Claude Code.

**Opus thinks. Sonnet executes. Haiku validates.**

## What It Does

`/yolo-story` runs the full BMAD implementation cycle in a loop with tiered AI agents:

```
Opus creates exhaustive stories -> Sonnet implements exactly as told ->
Haiku validates build/tests -> Sonnet reviews adversarially ->
Sonnet fixes issues (resumed session) -> Commit -> Next Story
```

It creates a git branch, implements every story in the epic, and pushes clean commits.
You start it, go to sleep, wake up to a completed epic.

`/yolo-story-hitl` does the same thing but sends detailed Telegram notifications after every step via n8n. Full implementation changelogs, not summaries.

`/yolo-revert` safely discards the yolo branch and returns to main.

## Agent Architecture

| Agent | Model | Role |
|-------|-------|------|
| Orchestrator | Opus | Extended thinking. Routes, decides, recovers |
| yolo-story-creator | Opus | Creates stories so exhaustive Sonnet can't drift |
| yolo-developer | Sonnet | Junior dev. Follows story exactly. TDD. No decisions |
| yolo-reviewer | Sonnet | Adversarial review. Fresh context every time |
| yolo-validator | Haiku | Build pass? Tests pass? Pure verification |
| yolo-health-checker | Haiku | Sprint-status vs git vs filesystem consistency |

### Session Reuse

The developer agent is **resumed** for fix cycles (saves ~50k tokens per round). The reviewer always gets **fresh context** (honest, unbiased review). Haiku agents are stateless.

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
/yolo-story       # Autonomous implementation (silent)
/yolo-story-hitl  # Autonomous + Telegram notifications
/yolo-revert      # Discard yolo branch, return to main
```

## HITL Telegram Notifications

`/yolo-story-hitl` sends detailed notifications to Telegram via an n8n webhook after every story step.

### Setup

1. **Send any message to [t.me/@yolostory_bot](https://t.me/@yolostory_bot)** to activate the bot
2. **Get your Telegram chat ID** (numeric, not username)
3. **Run `/yolo-story-hitl`** — it will ask for your chat ID

The default webhook is `https://auto.totalhumandesign.com/webhook/yolo-story`. To self-host, import `n8n/yolo-telegram-webhook.json` into your own n8n instance and provide the custom URL when prompted.

### What You Get

Every step sends a full implementation changelog:

- **Story Created** — all tasks, all acceptance criteria, branch info
- **Story Implemented** — every file created/modified with descriptions, dependencies with versions, test counts
- **Review Findings** — every finding with file:line, severity, description
- **Fix Cycle** — every fix applied, files modified, new tests
- **Story Done** — commit hash, final state, key decisions, human-check items
- **Story Flagged** — unresolved issues, human decisions needed
- **Epic Complete** — per-story summary, totals, merge instructions

Notifications are fire-and-forget. They never pause execution. If n8n is down, yolo continues silently.

### Privacy

`sprint-status.yaml` contains your Telegram chat ID. The skill automatically ensures it's in `.gitignore` before any commits.

## How It Works

1. Reads `_bmad/bmm/config.yaml` to find project artifacts
2. Reads `sprint-status.yaml` to find the next story
3. Creates `yolo/{epic-name}` git branch
4. For each story:
   - **Health Check (Haiku):** Validates branch, working tree, remote sync, sprint-status consistency
   - **Create Story (Opus):** Follows `create-story` workflow. Exhaustive analysis, zero ambiguity
   - **Implement (Sonnet):** Follows `dev-story` workflow. TDD, follows story exactly
   - **Validate (Haiku):** Gate check — build and tests must pass before review
   - **Review (Sonnet):** Independent adversarial review, min 10 findings
   - **Fix (Sonnet, resumed):** Fixes all findings, re-validates, re-reviews (max 3 rounds)
   - **Commit:** Atomic commit per story, push to remote
5. Loops until epic is complete

## Safety

- All work on a `yolo/{epic-name}` branch — main is never touched
- Opus creates stories. Sonnet follows them. Separation prevents drift
- Code review in separate subagent context for honest, unbiased review
- Haiku validates build/tests as a gate before review
- Each story is an atomic commit — easy to cherry-pick or revert
- Max 3 fix-review rounds — stories get flagged, never loop infinitely
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
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   ├── yolo-story/
│   │   ├── SKILL.md              # Autonomous loop (silent)
│   │   ├── workflow-guide.md     # BMAD workflow paths and config resolution
│   │   ├── state-machine.md      # Sprint status transitions, subagent tracking
│   │   └── compact-anchor.md     # Recovery after context compaction
│   ├── yolo-story-hitl/
│   │   └── SKILL.md              # Same loop + Telegram notifications
│   └── yolo-revert/
│       └── SKILL.md              # Branch revert
├── agents/
│   ├── yolo-reviewer.md          # Adversarial code reviewer (Sonnet)
│   ├── yolo-story-creator.md     # Story architect (Opus)
│   ├── yolo-developer.md         # Junior developer (Sonnet)
│   ├── yolo-validator.md         # QA validation bot (Haiku)
│   └── yolo-health-checker.md    # State consistency checker (Haiku)
├── n8n/
│   └── yolo-telegram-webhook.json  # Importable n8n workflow
└── README.md
```

## License

MIT
