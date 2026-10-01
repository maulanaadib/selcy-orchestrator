# selcy-orchestrator

The automation layer for [skills-selcy](https://github.com/maulanaadib/skills-selcy). A Hermes profile plus one skill that turns an approved spec bundle into a built product with no further human attention.

## What it does

`skills-selcy` defines the loop: `/sdd-selcy` writes the spec bundle once, then each feature is built in its own chat with `/selcy`. That loop is safe because every decision lives in files, not in chat memory.

This repo replaces the human in that loop. After you approve the plan, the orchestrator reads `context/progress-tracker.md`, creates one kanban card per remaining feature, and the dispatcher keeps spawning the orchestrator profile to work them until every feature is done or something needs you.

## The trigger

This is the part most people get wrong, so it is stated plainly: **there is no cron job, no webhook, and no human starting each feature. The kanban dispatcher is the trigger.**

The dispatcher is a loop inside the Hermes gateway that ticks every 60 seconds. On every tick it claims any card in `ready` status and spawns the assigned profile as a worker. So the whole automation is: create the cards once, then walk away. The board is the event loop.

```text
/sdd-selcy  →  you approve  →  /selcy-orchestrate (once)
                                    │
                       ┌────────────┴────────────┐
                       │ one kanban card per     │
                       │ feature, chained in     │
                       │ order                   │
                       └────────────┬────────────┘
                                    │
                       ┌────────────┴────────────┐
                       │ dispatcher ticks 60s    │ ← already running,
                       │ claims next ready card  │   survives reboot
                       │ spawns the worker       │
                       └────────────┬────────────┘
                                    │
                       ┌────────────┴────────────┐
                       │ delegate one feature    │ ← opencode run
                       │ verify Check When Done  │   (new session,
                       │ complete or block       │    visible in history)
                       └─────────────────────────┘
                                    │
                         stop: owed decision / failed / gate mismatch
```

## The pieces

| Piece | Job | Where it lives |
| --- | --- | --- |
| `selcy-orchestrator` Hermes profile | Reads the card, delegates one feature, verifies, closes the card | `~/.hermes/profiles/selcy-orchestrator/` (created by you, see the runbook) |
| `SOUL.md` | The worker's rules: never build, never edit specs, never answer owed decisions | Same folder |
| `selcy-orchestrate` skill | Two phases: bootstrap the board, then the worker loop | `skills/selcy-orchestrate/SKILL.md` in this repo |
| Kanban board | The trigger and the state. One card per feature, chained in order | `~/.hermes/kanban.db` |
| Gateway dispatcher | The event loop. Claims a ready card every 60s and spawns the worker | Runs as a background service (Windows startup item, or systemd unit on Linux) |

## Install

Uses [npx skills](https://github.com/vercel-labs/skills), into the project you want to build:

```bash
# The spec-driven skills (the builder's tools)
npx skills@latest add maulanaadib/skills-selcy -a opencode

# This orchestrator
npx skills@latest add maulanaadib/selcy-orchestrator -a opencode
```

Both are needed. The orchestrator delegates to a builder that uses the `skills-selcy` skills.

Then three one-time setup steps on the Hermes profile — install the skill into the profile, enable the kanban toolset, and confirm the dispatcher is running. Full walkthrough: **[docs/orchestrator-runbook.md](docs/orchestrator-runbook.md)**.

## How to use it

Quick version:

```bash
# 1. Interview (you are present)
cd <new-project-dir>
opencode run "/sdd-selcy"
# ...answer the panels, approve the plan...

# 2. Bootstrap the board (once)
hermes -p selcy-orchestrator chat
/selcy-orchestrate

# 3. Build (you are not)
hermes kanban watch
```

After step 2 you can close the chat. The dispatcher takes over within 60 seconds.

## The three stop conditions

Unattended does not mean unsupervised. The orchestrator blocks the card and reports when the contract cannot answer a question:

| Kind | What happened | What it surfaces |
| --- | --- | --- |
| **Owed decision** | The builder asked a question the bundle does not answer | The question verbatim. It never answers it, never picks the recommended option on your behalf |
| **Failed** | The build errored | The error and the spec section it was working on |
| **Gate mismatch** | The builder claimed done but a `Check When Done` item is unaddressed, or the tracker did not advance | Which item |

A blocked card stops the rest of the chain automatically, because each card only promotes when its parent is done. Resuming is `hermes kanban unblock <card-id>` — nothing restarts, the next tick picks it up.

## Why this is separate from skills-selcy

`skills-selcy` is the spec-driven method: seven skills, each a distinct job, usable by any agent, with no automation assumptions. This repo is automation for one specific setup: Hermes orchestrating opencode. Coupling them would force everyone who installs `skills-selcy` to also take a Hermes profile and an opencode delegation contract they may not want.

The dependency runs one way: this repo depends on `skills-selcy`, never the reverse.

## Requirements

- A Hermes profile named `selcy-orchestrator` (the runbook creates it).
- The kanban toolset enabled on that profile (`hermes -p selcy-orchestrator tools enable kanban`).
- The Hermes gateway running. On Windows it installs as a startup item; on Linux `hermes gateway start` installs a systemd user unit. Check with `hermes gateway status`.
- `opencode` on PATH, for the delegation.
- `git` on PATH, for kanban mode.
- A `context/` bundle written by `/sdd-selcy`. Without one, the orchestrator refuses and tells you to run it first.

## License

MIT
