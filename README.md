# selcy-orchestrator

The automation layer for [skills-selcy](https://github.com/maulanaadib/skills-selcy). A Hermes profile plus one skill that drives the whole spec-driven build loop unattended.

## What it does

`skills-selcy` defines the loop: `/sdd-selcy` writes the spec bundle once, then each feature is built in its own chat with `/selcy`. That loop is safe because every decision lives in files, not in chat memory.

This repo replaces the human in that loop. After you approve the plan, the orchestrator reads `context/progress-tracker.md`, delegates one feature at a time to `opencode run`, verifies the result against the spec's `Check When Done`, and advances to the next feature. It stops only when something needs you.

```text
/sdd-selcy  →  you approve  →  /selcy-orchestrate
                                    │
                       ┌────────────┴───────────┐
                       │ read progress-tracker  │
                       │ delegate one feature   │ ← opencode run (new session, visible in your history)
                       │ verify Check When Done │
                       │ advance, loop          │
                       └────────────────────────┘
                                    │
                         stop: owed decision / failed / gate mismatch
```

## The pieces

| Piece | Job | Where it lives |
| --- | --- | --- |
| `selcy-orchestrator` Hermes profile | Reads the tracker, delegates, verifies, advances | `~/.hermes/profiles/selcy-orchestrator/` (created by you, see the runbook) |
| `SOUL.md` | The orchestrator's rules: never build, never edit specs, never answer owed decisions | Same folder |
| `selcy-orchestrate` skill | The loop itself, step by step, plus its stop conditions | `skills/selcy-orchestrate/SKILL.md` in this repo |

## Install

Uses [npx skills](https://github.com/vercel-labs/skills), into the project you want to build:

```bash
# The spec-driven skills (the builder's tools)
npx skills@latest add maulanaadib/skills-selcy -a opencode

# This orchestrator
npx skills@latest add maulanaadib/selcy-orchestrator -a opencode
```

Both are needed. The orchestrator delegates to a builder that uses the `skills-selcy` skills.

## How to use it

Full walkthrough, including the Hermes profile setup, kanban mode, and how to resume after a stop: **[docs/orchestrator-runbook.md](docs/orchestrator-runbook.md)**.

Quick version:

```bash
# 1. Interview (you are present)
cd <new-project-dir>
opencode run "/sdd-selcy"
# ...answer the panels, approve the plan...

# 2. Build (you are not)
hermes -p selcy-orchestrator chat
/selcy-orchestrate
```

## The three stop conditions

Unattended does not mean unsupervised. The orchestrator stops and reports when the contract cannot answer a question:

| Kind | What happened | What it surfaces |
| --- | --- | --- |
| **Owed decision** | The builder asked a question the bundle does not answer | The question verbatim. It never answers it, never picks the recommended option on your behalf |
| **Failed** | The build errored | The error and the spec section it was working on |
| **Gate mismatch** | The builder claimed done but a `Check When Done` item is unaddressed, or the tracker did not advance | Which item |

A single feature failure stops the loop, because the next feature may be built on a broken foundation.

## Why this is separate from skills-selcy

`skills-selcy` is the spec-driven method: seven skills, each a distinct job, usable by any agent, with no automation assumptions. This repo is automation for one specific setup: Hermes orchestrating opencode. Coupling them would force everyone who installs `skills-selcy` to also take a Hermes profile and an opencode delegation contract they may not want.

The dependency runs one way: this repo depends on `skills-selcy`, never the reverse.

## Requirements

- A Hermes profile named `selcy-orchestrator` (the runbook creates it).
- `opencode` on PATH, for the delegation.
- `git` on PATH, for kanban mode.
- A `context/` bundle written by `/sdd-selcy`. Without one, the orchestrator refuses and tells you to run it first.

## License

MIT
