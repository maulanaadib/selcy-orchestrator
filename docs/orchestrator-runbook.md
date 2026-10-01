# Runbook: the selcy-orchestrator automation

How to go from an empty folder to a built product, unattended, with Hermes as the orchestrator and opencode as the builder.

## What this is

```text
hermes (selcy-orchestrator profile)        ← the orchestrator, reads progress-tracker.md
   │
   └─ opencode run "build feature NN"      ← the builder, one feature per session
         │
         └─ context/progress-tracker.md    ← the only source of truth
```

Three pieces, each with one job:

| Piece | Job | Where it lives |
| --- | --- | --- |
| `selcy-orchestrator` profile | Read the tracker, delegate one feature, verify, advance | `C:\Users\ADIB\AppData\Local\hermes\profiles\selcy-orchestrator\` |
| `SOUL.md` | The orchestrator's rules: never build, never edit specs, never answer owed decisions | Same folder |
| `selcy-orchestrate` skill | The loop itself, step by step | This repo, `skills/selcy-orchestrate/SKILL.md` |

This repo is separate from `skills-selcy`. `skills-selcy` holds the seven spec-driven skills (`/sdd-selcy`, `/selcy`, `/selcy-check`, `/selcy-test`, `/selcy-debug`, `/selcy-sync`). This repo holds only the automation that drives them.

## One time setup

### 1. Install both skill bundles into the new project

The project needs the spec-driven skills (the builder's tools) and this orchestrator skill:

```bash
cd <new-project-dir>
npx skills@latest add maulanaadib/skills-selcy -a opencode
npx skills@latest add maulanaadib/selcy-orchestrator -a opencode
```

### 2. Point the orchestrator profile at the project

From the project directory:

```bash
hermes -p selcy-orchestrator skills trust .
```

### 3. Confirm the profile can run

```bash
hermes -p selcy-orchestrator profile show
```

Check it has a model and API keys. If not, run `selcy-orchestrator setup` (the wrapper the profile create step made) or set the keys in the profile's `.env`.

## The run

### Phase 1: the interview (you are present)

```bash
cd <new-project-dir>
opencode run "/sdd-selcy"
```

You answer the panels. You approve the plan at the confirmation panel. The bundle is written to `context/`.

**This is the last time you must be present.** Everything after this is the orchestrator.

### Phase 2: the build (you are not)

Pick one of the two modes.

**Direct mode** — you watch every delegation:

```bash
hermes -p selcy-orchestrator chat
```

Then in the chat:

```text
/selcy-orchestrate
```

**Kanban mode** — durable, survives a closed chat, failure count on the board:

```bash
hermes kanban create "Build all features in the approved spec bundle" `
  --assignee selcy-orchestrator `
  --workspace dir:<project-dir> `
  --skill selcy-orchestrate `
  --max-retries 1 `
  --goal

hermes kanban daemon --interval 60
```

On Windows PowerShell, use backtick `` ` `` for line continuation, not backslash.

The daemon polls every 60 seconds, picks up the task, and the orchestrator runs the loop: read tracker, delegate one feature to `opencode run`, verify against `Check When Done`, advance, repeat.

### Phase 3: when it stops

It stops for one of three reasons. Each has a different response:

| Why it stopped | What you do |
| --- | --- |
| **Owed decision** — the builder asked a question the bundle does not answer | Answer it. Then edit `context/` if the answer is a new rule, and resume with `/selcy-orchestrate`. |
| **Failed** — the build errored | Read the opencode session for the error. Fix the environment or the code, then resume. Do not blindly re-run. |
| **Gate mismatch** — the builder claimed done but a checklist item is unaddressed | Read the opencode session. Either the item is genuinely missing, or the spec is wrong. Fix one of them, then resume. |

Resuming is the same command both modes: `/selcy-orchestrate`. It reads `progress-tracker.md` first, so it picks up wherever the loop stopped.

## How to watch it work

Every `opencode run` creates a session you can inspect later:

```bash
opencode session list
opencode export <session-id>
```

So the full history of every feature build is visible to you, even in kanban mode where you were not watching.

For kanban mode, the board shows the state:

```bash
hermes kanban list
hermes kanban show <task-id>
```

## The rules that keep this safe

These are in `SOUL.md` and in the skill. They are the reason unattended is safe here:

- **One feature per delegation, always.** A failure stays contained to one feature.
- **The orchestrator never writes application code.** It delegates. If it could build, it could build the wrong thing at full speed.
- **The orchestrator never edits `context/`.** Those are contracts. A wrong spec is a human decision.
- **The orchestrator never answers an owed decision.** It surfaces the question verbatim and stops. It never picks the recommended option on your behalf.
- **A failed build is never auto-retried.** `--max-retries 1`. A retry without new information is the same failure again.
- **Green output is not proof.** The orchestrator verifies against the spec's `Check When Done` and re-reads the tracker to confirm the builder itself advanced it.
- **Progress is in the file, not in the agent.** The orchestrator forgets between turns and reads `progress-tracker.md` at the start of every turn.

## When not to use this

- **You want to review each feature before the next one starts.** This is unattended. Use `/selcy` per feature in a new chat instead.
- **The project has no spec bundle.** Run `/sdd-selcy` first. The orchestrator will refuse and tell you to.
- **You want to change what is being built.** Edit the bundle, run `/selcy-sync`, then resume. The orchestrator never edits the bundle for you.

## Uninstall or disable

```bash
# Stop the daemon
hermes kanban daemon --stop    # or kill the process holding the pidfile

# Remove the profile
hermes profile delete selcy-orchestrator

# Remove the skill from a project
npx skills@latest remove selcy-orchestrate
```

The `context/` bundle stays untouched. Removing the orchestrator never removes the specs or the code.
