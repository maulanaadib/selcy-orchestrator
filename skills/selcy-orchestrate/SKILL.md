---
name: selcy-orchestrate
allowed-tools: Bash, Read, Grep, Glob, Write, Edit, Agent
description: "Run /selcy-orchestrate after /sdd-selcy and you approve the plan. Reads progress-tracker.md, delegates one feature at a time to `opencode run`, verifies the result against the spec's Check When Done, and advances. Unattended: it loops through every feature and stops only on an owed decision or a failed gate."
---

## What this skill does

The orchestrator. It turns an approved spec bundle into a built product without further human attention.

`/sdd-selcy` writes the bundle. You approve the plan. Then this skill runs the whole build loop: it reads `context/progress-tracker.md`, delegates the current feature to `opencode run`, verifies the result, and advances to the next feature. It does this until every feature is complete, or until something needs you.

It exists because the skills-selcy loop is deliberately one feature per chat, and a human opening a new chat per feature is the bottleneck on a seven feature project. The loop's safety does not come from the human being present each time; it comes from the contract being in the files. This skill drives that loop mechanically, and stops the moment the contract cannot answer a question.

## The two ways to run it

- **Direct.** `hermes -p selcy-orchestrator chat` with `/selcy-orchestrate`. You see every delegation, every result, every stop. Interactive, durable only for the session.
- **Kanban.** A task on the kanban board, picked up by `hermes kanban daemon`. Durable: the daemon retries on crash, blocks after repeated failure, and the task survives a reboot. Use this when the build is long enough that you want to walk away.

Both run the same loop below. The kanban path just wraps it in a task lifecycle.

## Prerequisites

All three must be true, or the skill stops and says which one is missing:

1. A `context/` bundle exists in the project directory, written by `/sdd-selcy`.
2. `context/progress-tracker.md` names a current goal that is a feature spec in `context/feature-specs/`.
3. `opencode` is on PATH and can run in the project directory.

The skill never creates a bundle, never edits the feature list, and never runs `/sdd-selcy`. If the bundle is missing, that is a human decision about what to build; tell the engineer to run `/sdd-selcy` first.

## The loop

Read this as the contract the orchestrator must not deviate from.

### Step 0: Confirm the gate state

Read `context/progress-tracker.md`. Find:

- **Current Phase** — if it says the build is complete, stop and report that.
- **Current Goal** — the feature to delegate. It must name a spec file.
- **Completed** — features already done. Never delegate one of these.
- **In Progress** — if a feature is listed here, resume it rather than starting fresh.

If Current Goal is missing or names no spec file, stop. That is a bundle state a human must fix.

### Step 1: Delegate one feature

Run `opencode run` in the project directory. The message names the feature spec and nothing else:

```bash
opencode run --dir <project-dir> "Read context/AGENTS.md, then build the feature in context/feature-specs/<NN-name>.md"
```

Rules that make this safe:

- **The message is a pointer, not a brief.** Do not paraphrase the spec, do not add requirements, do not scope it. The builder reads the spec; your summary can only lose information.
- **One feature per run.** Never two. A failure must stay contained to one feature.
- **`--dir` pins the working directory.** The orchestrator may be running from elsewhere; the build must happen in the project directory.
- **Wait for it.** `opencode run` is synchronous. It returns when the build is done or when it fails. Do not background it, do not poll, do not time it out and move on.

### Step 2: Read the result

`opencode run` exits zero or non-zero. Both are information:

- **Exit zero** → the builder believes it finished. That is a claim, not proof. Go to Step 3.
- **Exit non-zero** → the build failed, or the builder stopped on a question it could not answer. Read the output. Go to Step 4.

### Step 3: Verify against the spec

Do not trust green output. Check it:

1. Read the feature spec's `## Check When Done`. Every item is a claim the build must prove.
2. Read the builder's output. Did it address each item, or only the ones that went well?
3. Read `context/progress-tracker.md` again. Did the builder move this feature to Completed and set the next Current Goal? If it did not, the builder itself did not consider the feature done; do not override that judgment.

If every checklist item is addressed and the tracker advanced → the feature is done. Loop to Step 0 for the next feature.

If an item was skipped, failed, or the tracker did not advance → the feature is not done. Go to Step 4.

### Step 4: Stop and report

The orchestrator never retries a failed build blindly, never answers an owed decision, and never edits the spec to make a failure go away. It stops and reports:

- **The feature** that stopped.
- **Why**, in one of three kinds:
  - **Failed** — the build errored. Surface the error and the spec section it was working on.
  - **Owed decision** — the builder asked a question the bundle does not answer. Surface the question verbatim. Do not answer it, do not pick the recommended option on the engineer's behalf.
  - **Gate mismatch** — the builder claimed done but a `Check When Done` item is unaddressed, or the tracker did not advance. Surface which item.
- **The exact command** to resume once the engineer has resolved it.

Then stop. Do not delegate the next feature. A failed feature means the next one may be built on a broken foundation.

## What this skill never does

- Never writes application code. It delegates; it does not build.
- Never edits a file under `context/`. Those are contracts. A wrong spec is a human decision.
- Never answers an owed decision. It surfaces the question and stops.
- Never batches features. One per delegation, always.
- Never retries a failed delegation automatically. A retry without new information is the same failure again.
- Never marks a feature complete. It reads the tracker; the builder writes it.
- Never runs `/sdd-selcy`, `/selcy`, `/selcy-check`, `/selcy-test`, or `/selcy-debug` itself. Those are the builder's tools, and the orchestrator delegating to itself is the loop it exists to prevent.

## When to stop the whole run, not just one feature

A single feature failure stops the loop. These stop the whole run and tell the engineer to rebuild the bundle:

- `progress-tracker.md` is missing or unreadable. The orchestrator has no source of truth.
- The Current Goal names a spec file that does not exist.
- Three consecutive features failed at delegation before any of them produced output. That pattern means the environment, not the features, is broken.

## Kanban mode

When running as a kanban task rather than direct chat, the loop is identical. The task lifecycle adds:

- **`--workspace dir:<project-dir>`** on `hermes kanban create` pins the build directory.
- **`--skill selcy-orchestrate`** loads this skill into the worker.
- **`--max-retries 1`** — a failed build must not auto-retry. The failure is reported, not retried, per Step 4.
- **`--goal`** lets the orchestrator keep going across turns until the tracker says complete or it hits a stop condition.

Create the task once, after you approve the plan:

```bash
hermes kanban create "Build all features in the approved spec bundle" \
  --assignee selcy-orchestrator \
  --workspace dir:<project-dir> \
  --skill selcy-orchestrate \
  --max-retries 1 \
  --goal
```

Then the daemon picks it up:

```bash
hermes kanban daemon --interval 60
```

The daemon is the durable path: it survives a chat closing, and the task's failure count is visible on the board. But it does not change the loop. Same steps, same stops, same refusal to retry a failure.

## Portability

Any Agent Skills client on macOS, Linux, Windows. `opencode` and `git` are the required CLIs. Shell snippets are POSIX reference; on Windows run them through the agent's cross platform shell. The delegation command is the only shell call that must work exactly; substitute the equivalent non-interactive invocation for your builder if it is not opencode.
