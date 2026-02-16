<p align="center">
  <strong>bmad-yolo</strong><br>
  <em>Autonomous sprint execution for BMAD projects</em>
</p>

<p align="center">
  <a href="#installation">Installation</a> &bull;
  <a href="#quick-start">Quick Start</a> &bull;
  <a href="#commands">Commands</a> &bull;
  <a href="#telegram-notifications">Telegram</a> &bull;
  <a href="#architecture">Architecture</a>
</p>

---

A [Claude Code](https://claude.ai/code) plugin that runs your entire BMAD Phase 4 (Implementation) autonomously. It creates stories, writes code with TDD, runs adversarial code reviews, fixes every finding, commits, and moves to the next story — until the epic is done.

Start it. Walk away. Come back to a completed epic on a clean branch.

## Installation

Add the marketplace and install:

```bash
/plugin marketplace add sphinxcode/bmad-yolo
/plugin install bmad-yolo@sphinxcode-bmad-yolo
```

Or via CLI outside a session:

```bash
claude plugin install bmad-yolo@sphinxcode-bmad-yolo --scope user
```

> **Scope options:** `user` (all projects), `project` (shared with collaborators), `local` (just you, this repo)

### Prerequisites

- [Claude Code](https://claude.ai/code) with plugin support
- A BMAD project with `_bmad/` installed (`npx bmad-method install`)
- Phases 1-3 complete: Brief, PRD, Architecture, Epics
- Sprint planning done (`sprint-status.yaml` populated)
- Git repo with a remote

## Quick Start

```
/yolo-story
```

That's it. The plugin reads your sprint-status, creates a `yolo/{epic-name}` branch, and starts executing stories.

## Commands

| Command | Description |
|---------|-------------|
| `/yolo-story` | Run the full sprint autonomously (silent) |
| `/yolo-story-hitl` | Same thing + Telegram notifications after every step |
| `/yolo-revert` | Discard the yolo branch and return to main |

## How It Works

```
for each story in sprint:

  1. Health Check      →  verify branch, working tree, sprint-status consistency
  2. Create Story      →  exhaustive story file with tasks, ACs, dev notes
  3. Implement         →  TDD per task (red → green → refactor)
  4. Validate          →  build passes, tests pass, tasks complete
  5. Review            →  independent adversarial code review (10+ findings)
  6. Fix               →  address all findings, re-validate, re-review
  7. Commit & Push     →  atomic commit per story on yolo/{epic} branch

  repeat until epic complete
```

Each story is one atomic commit. The branch is never merged automatically — that's your call after reviewing.

## Architecture

Six specialized agents with strict model separation:

```
OPUS (thinks)                    SONNET (executes)              HAIKU (validates)
├── Orchestrator                 ├── Developer                  ├── Validator
│   routes, decides, recovers   │   follows story exactly       │   build/test/lint gate
│                                │   TDD, no decisions           │
└── Story Creator                └── Reviewer                   └── Health Checker
    exhaustive analysis              adversarial, fresh context      state consistency
    zero ambiguity                   min 10 findings
```

| Agent | Model | Spawning | Purpose |
|-------|-------|----------|---------|
| Orchestrator | Opus | Main session | Extended thinking for routing, recovery, edge cases |
| Story Creator | Opus | Per story | Creates stories so detailed Sonnet can't drift |
| Developer | Sonnet | Per story, **resumed** for fixes | Follows story file exactly. TDD. No independent decisions |
| Reviewer | Sonnet | Per review, **always fresh** | Adversarial review in separate context. Honest by design |
| Validator | Haiku | Per validation | Build pass? Tests pass? File count matches? Zero creativity |
| Health Checker | Haiku | Per story | Branch, working tree, remote sync, sprint-status vs files |

**Token optimization:** The Developer agent is resumed for fix cycles (~50k token savings per round). The Reviewer always gets fresh context to prevent anchoring bias.

## Telegram Notifications

`/yolo-story-hitl` sends full implementation changelogs to Telegram after every step via n8n. Not summaries — complete changelogs with every file, every finding, every fix.

### Setup

1. Message [t.me/@yolostory_bot](https://t.me/@yolostory_bot) to activate the bot
2. Get your numeric Telegram chat ID
3. Run `/yolo-story-hitl`

Default webhook: `https://auto.totalhumandesign.com/webhook/yolo-story`

### Events

| Event | What you receive |
|-------|-----------------|
| Story Created | Title, all tasks, all acceptance criteria, branch |
| Implemented | Every file created/modified, dependencies, test counts |
| Review Findings | Every finding with `file:line`, severity, description |
| Fix Cycle | Every fix applied, files changed, new tests |
| Story Done | Commit hash, final stats, key decisions, human-check items |
| Story Flagged | Unresolved issues, what was tried, decisions needed |
| Epic Complete | Per-story breakdown, totals, merge instructions |

Notifications are fire-and-forget. They never block execution.

### Self-Hosting

Import `n8n/yolo-telegram-webhook.json` into your own n8n instance, configure the Telegram bot token, and provide your custom webhook URL when prompted.

### Privacy

`sprint-status.yaml` stores your Telegram chat ID. The plugin ensures it's in `.gitignore` before any commit.

## Safety

| Guarantee | How |
|-----------|-----|
| Main branch untouched | All work on `yolo/{epic-name}` branch |
| No AI drift | Opus writes stories, Sonnet follows them. Separation of concerns |
| Honest code review | Reviewer runs in fresh context — never sees implementation reasoning |
| No infinite loops | Max 3 fix-review rounds, then flag and move on |
| Easy rollback | One atomic commit per story. `/yolo-revert` to discard everything |
| No destructive ops | DB drops, deployments, CI triggers are logged but never executed |

### What it will never do

- Merge into main
- Force push or rebase
- Execute destructive database operations
- Deploy to any environment
- Run interactive flows (OAuth, browser prompts)

## Plugin Structure

```
bmad-yolo/
├── .claude-plugin/
│   ├── plugin.json                     # Plugin manifest (v1.1.0)
│   └── marketplace.json                # Self-hosted marketplace config
├── skills/
│   ├── yolo-story/
│   │   ├── SKILL.md                    # Autonomous loop (silent)
│   │   ├── workflow-guide.md           # BMAD workflow paths & config resolution
│   │   ├── state-machine.md            # Status transitions & subagent tracking
│   │   └── compact-anchor.md           # Recovery after context compaction
│   ├── yolo-story-hitl/
│   │   └── SKILL.md                    # Autonomous loop + Telegram notifications
│   └── yolo-revert/
│       └── SKILL.md                    # Branch revert & cleanup
├── agents/
│   ├── yolo-story-creator.md           # Story architect (Opus)
│   ├── yolo-developer.md               # Implementation agent (Sonnet)
│   ├── yolo-reviewer.md                # Adversarial code reviewer (Sonnet)
│   ├── yolo-validator.md               # Build/test gate (Haiku)
│   └── yolo-health-checker.md          # State consistency checker (Haiku)
├── n8n/
│   └── yolo-telegram-webhook.json      # Importable n8n workflow
└── README.md
```

## Related

- [BMAD-METHOD](https://github.com/bmad-method/BMAD-METHOD) — The framework this plugin automates
- [Claude Code Plugins](https://docs.anthropic.com/en/docs/claude-code/plugins) — Plugin documentation

## License

MIT
