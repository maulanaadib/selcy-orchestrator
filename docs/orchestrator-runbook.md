# Runbook: the selcy-orchestrator automation

How to go from an empty folder to a built product, unattended, with Hermes as the orchestrator and opencode as the builder.

## What this is

```text
you  ──/sdd-selcy──▶  context/ bundle  ──you approve──▶  /selcy-orchestrate (once)
                                                            │
                                                            ▼
                                              kanban board: one card per feature
                                                            │
                                              ┌─────────────┴──────────────┐
                                              │  gateway dispatcher         │  ← ticks every 60s,
                                              │  claims the next ready card  │     already running
                                              │  spawns selcy-orchestrator   │
                                              └─────────────┬──────────────┘
                                                            │
                                       ┌────────────────────┴───────────────┐
                                       │ worker: delegate one feature       │
                                       │   to `opencode run`                │ ← new session,
                                       │ verify against Check When Done     │   visible in your history
                                       │ kanban_complete / kanban_block     │
                                       └────────────────────────────────────┘
                                                            │
                                              stop: owed decision / failed / gate mismatch
```

Five pieces, each with one job:

| Piece | Job | Where it lives |
| --- | --- | --- |
| `selcy-orchestrator` profile | Read the card, delegate one feature, verify, close the card | `~/.hermes/profiles/selcy-orchestrator/` |
| `SOUL.md` | The worker's rules: never build, never edit specs, never answer owed decisions | Same folder |
| `selcy-orchestrate` skill | The two phases: bootstrap the board, then the worker loop | This repo, `skills/selcy-orchestrate/SKILL.md` |
| Kanban board | The trigger and the state. One card per feature, chained in order | `~/.hermes/kanban.db` |
| Gateway dispatcher | The event loop. Claims a ready card every 60s and spawns the worker | Runs as a background service |

The trigger is not a cron job and not a human. It is the dispatcher: a loop inside the gateway that claims any card in `ready` status and spawns the assigned profile. You create the cards once; the board does the rest.

This repo is separate from `skills-selcy`. `skills-selcy` holds the spec-driven skills (`/sdd-selcy`, `/selcy`, `/selcy-check`, `/selcy-test`, `/selcy-debug`, `/selcy-sync`). This repo holds only the automation that drives them.

## One time setup

### 1. Install both skill bundles into the new project

The project needs the spec-driven skills (the builder's tools) and this orchestrator skill:

```bash
cd <new-project-dir>
npx skills@latest add maulanaadib/skills-selcy -a opencode
npx skills@latest add maulanaadib/selcy-orchestrator -a opencode
```

### 2. Install the orchestrator skill into the Hermes profile

The worker needs the skill on the profile, not just in the project:

```bash
hermes -p selcy-orchestrator skills install https://raw.githubusercontent.com/maulanaadib/selcy-orchestrator/main/skills/selcy-orchestrate/SKILL.md --yes
```

### 3. Enable the kanban toolset on the profile

The worker closes its own card with `kanban_complete` / `kanban_block`. Without this toolset those tools are not in its schema and it cannot finish a card:

```bash
hermes -p selcy-orchestrator tools enable kanban
```

### 4. Confirm the profile can run

```bash
hermes -p selcy-orchestrator profile show
```

Check it has a model and API keys. If not, run `selcy-orchestrator setup` (the wrapper the profile create step made) or set the keys in the profile's `.env`.

### 5. Confirm the dispatcher is running

It should already be up — on Windows it installs as a startup item, on Linux `hermes gateway start` installs a systemd user unit. Verify:

```bash
hermes gateway status
hermes cron status
```

If it is not running, start it and it will survive reboots:

```bash
hermes gateway start
```

On Linux, one extra step matters. Workers are fire-and-forget processes that must outlive the dispatcher tick, so the gateway spawns each one in its own systemd scope. That needs your user's session bus — without it the spawn is refused as an infrastructure failure and the card stays in `ready`. Enable lingering once:

```bash
sudo loginctl enable-linger <your-user>
```

This is the only Linux-specific gotcha in the whole setup. Everything else — the board, the dispatcher loop, the worker protocol, the skill — is identical on both platforms.

## The run

### Phase 1: the interview (you are present)

```bash
cd <new-project-dir>
opencode run "/sdd-selcy"
```

You answer the panels. You approve the plan at the confirmation panel. The bundle is written to `context/`.

**This is the last time you must be present.** Everything after this is the orchestrator.

### Phase 2: bootstrap the board (you do this once)

```bash
hermes -p selcy-orchestrator chat
```

Then in the chat:

```text
/selcy-orchestrate
```

The skill reads `context/progress-tracker.md`, creates one kanban card per remaining feature, chains them in order, and reports the card ids. Then it stops. It does not start working them — that is the dispatcher's job.

### Phase 3: the build (you are not)

Within 60 seconds the dispatcher claims the first card and spawns the orchestrator profile as a worker. The worker delegates that one feature to `opencode run`, verifies the result against the spec's `Check When Done`, and closes the card. Completing a card promotes the next one in the chain, and the next tick spawns the worker again.

You can watch it without touching it:

```bash
hermes kanban list              # the board
hermes kanban watch             # live event stream, Ctrl+C to exit
hermes kanban show <card-id>    # one card: comments, runs, events
hermes kanban log <card-id>     # the worker's own log
```

Every `opencode run` also creates a session you can inspect later, so the full history of every feature build is visible even if you were not watching:

```bash
opencode session list
opencode export <session-id>
```

### Phase 4: when it stops

It stops for one of three reasons. Each has a different response:

| Why it stopped | What you do |
| --- | --- |
| **Owed decision** — the builder asked a question the bundle does not answer | Answer it. Then edit `context/` if the answer is a new rule, and unblock the card. |
| **Failed** — the build errored | Read the opencode session for the error. Fix the environment or the code, then unblock. Do not blindly re-run. |
| **Gate mismatch** — the builder claimed done but a checklist item is unaddressed | Read the opencode session. Either the item is genuinely missing, or the spec is wrong. Fix one of them, then unblock. |

Resuming never restarts anything. The board is the state:

```bash
hermes kanban list --status blocked     # see what stopped and why
hermes kanban show <card-id>            # read the block reason
hermes kanban unblock <card-id>         # back to ready; next tick resumes it
```

The worker reads `progress-tracker.md` first, so it picks up wherever the loop stopped.

## The rules that keep this safe

These are in `SOUL.md` and in the skill. They are the reason unattended is safe here:

- **One feature per card, one card per feature.** A failure stays contained to one feature.
- **Cards are chained in order.** A card only promotes when its parent is done, so a blocked card stops the whole rest of the chain automatically.
- **The orchestrator never writes application code.** It delegates. If it could build, it could build the wrong thing at full speed.
- **The orchestrator never edits `context/`.** Those are contracts. A wrong spec is a human decision.
- **The orchestrator never answers an owed decision.** It surfaces the question verbatim and blocks. It never picks the recommended option on your behalf.
- **A failed build is never auto-retried.** The card stays blocked until a human unblocks it. A retry without new information is the same failure again.
- **Green output is not proof.** The worker verifies against the spec's `Check When Done` and re-reads the tracker to confirm the builder itself advanced it.
- **Progress is in the files, not in the agent.** The worker forgets between spawns and reads `progress-tracker.md` at the start of every run. The board carries the state, not the model.

## When not to use this

- **You want to review each feature before the next one starts.** This is unattended. Use `/selcy` per feature in a new chat instead.
- **The project has no spec bundle.** Run `/sdd-selcy` first. The orchestrator will refuse and tell you to.
- **You want to change what is being built.** Edit the bundle, run `/selcy-sync`, then resume. The orchestrator never edits the bundle for you.

## Uninstall or disable

```bash
# Stop the dispatcher
hermes gateway stop

# Remove the profile
hermes profile delete selcy-orchestrator

# Remove the skill from a project
npx skills@latest remove selcy-orchestrate
```

The `context/` bundle stays untouched. Removing the orchestrator never removes the specs or the code.
