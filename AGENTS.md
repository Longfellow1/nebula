<!-- BEGIN brain.md -->
## Project Brain

This project keeps a **Project Brain**: a persistent memory layer of its durable decisions, requirements, and constraints. Read `./BRAIN.md` for the full read/write contract.

Maintain the brain as part of normal coding work — not as a separate task. While discussing or implementing features:
- **Start of a task:** load relevant context with the `brain` CLI (`list-pages`, `read-page`, `read-root`). Prefer a narrow read over scanning everything.
- **When a decision, requirement, constraint, or durable insight settles** (in chat or while coding): capture it immediately via the `brain` CLI. Do not wait to be asked and do not batch it for later.
- **Pure implementation with no new decision:** do not write to the brain.
- **When overturning a prior conclusion:** update the page (`update-truth` and/or `append-timeline` with `kind: reversal`, or `archive-page`).
- Only store what will still matter in six months and is hard to reconstruct from the code alone.
- All reads and writes go through the `brain` CLI — never hand-edit brain files.

The brain skills (`brain-setup`, `brain-page`, `brain-ingest`, `brain-bootstrap`) are installed in your global skills directory. Prefer `brain init` to scaffold a new project.
<!-- END brain.md -->

## Current contribution context

This checkout is preparing a local implementation proposal for Nebula Issue #144
(Session timeline / replay). Load the Brain page `issue-144-session-timeline`
through the brain CLI, then read `P0_PLAN.md` for scope and progress. Evidence and
the draft event contract are in `ARCHITECTURE_GAP.md` and `EVENT_SCHEMA.md`.

On this machine, invoke the installed CLI from the Nebula root:

```sh
node /Users/Harland/.codex/skills/brain-page/bin/brain.mjs read-page issue-144-session-timeline
```

If the checkout moves to another machine, resolve the brain-page skill's
`bin/brain.mjs` there; this path is a local handoff convenience, not a runtime
dependency or an upstream packaging requirement.

The current task is documentation and context preservation. Parser, fixtures and
executable tests have not been implemented. Do not modify core protocol, database
migrations or Review UI, or post GitHub comments/open a PR. The user is arranging
worktree organization with agy; do not infer permission to switch branches,
stage, commit or clean the parent C_Agents project from these planning documents.

## [S7] Task history

- 2026-10-16: Read-only research at `a00b7607` distinguished the placeholder
  History tab, orchestrator PTY-tail read and generic transcript-first read.
  Initialized Brain and wrote three planning drafts. The standalone parser
  boundary and event contract remain proposals; implementation and Rust tests
  are not started. See the Brain project page for evidence and current status.
