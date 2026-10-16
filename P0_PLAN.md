# Issue #144: P0 plan and progress

Last updated: 2026-10-16 (Asia/Taipei).
Reviewed upstream commit: `a00b76075913bb098021177295e0dab37d3521aa`.
This is a local contribution proposal, not a maintainer-approved implementation.

## Current status

| Work | Status |
|---|---|
| Read README, architecture, orchestrator docs and Review code | Complete |
| Trace Claude transcript/hook ingress and History/context/PTY paths | Complete; source evidence in [ARCHITECTURE_GAP.md](ARCHITECTURE_GAP.md) |
| Propose internal event semantics, limits, privacy and extension seam | Drafted in [EVENT_SCHEMA.md](EVENT_SCHEMA.md); maintainer decisions open |
| Preserve background and handoff context | Brain initialized; contribution page and root roadmap populated |
| Produce the three planning documents | Drafts written; relative links, source-path/line bounds, code fences and inline JSONL syntax checked |
| Implement parser / create fixture files / add executable tests | **Not started**; inline example only, no parser-validation claim |
| Run Rust build/tests or validate real Claude version compatibility | **Not run**; no source changes in this planning iteration |
| Organize worktree, branch, staging or commits | User/agy responsibility; Codex has not performed these mutations |
| GitHub comment / PR | None; explicitly outside authorization |

Documentation verification: Brain `lint-links` passed for 1 project page and
6 root pages. Scope audit found exactly 14 expected new documentation/support
files, no tracked-file changes and an empty staging area. These checks establish
document consistency and scope, not runtime correctness or maintainer approval.

Original request: prepare a maintainer-friendly plan for per-session audit history
from harness transcripts/hooks, Claude first. The first turn was reconnaissance
and alignment only. The follow-up requests a Brain and necessary documentation
and progress updates; it does not authorize full parser/UI implementation.
The parent C_Agents checkout has unrelated local work. Nebula is a separate
nested clone; do not clean or stage the parent as part of this contribution.

## Proposed P0 acceptance boundary

A pure read-only parser accepts one explicitly supplied Claude JSONL source,
session identity, source generation and limits. It produces deterministic,
source-ordered events, redacted structural summaries, stable source references
and bounded diagnostics/coverage. It handles tool-use/result correlation without
assuming every user envelope is human input. It is exercised only against
invented, sanitized inputs in deterministic unit tests.

No core protocol changes, migrations, Review UI changes, background sweeps,
status behavior changes, raw transcript exports, global transcript discovery,
Codex/Pi adapters, command re-execution or historical diff reconstruction.
Do not silently change `ReadSession`, `SessionText`, `OrchestratorRead` or
`OutputTail` semantics. P0 adds no runtime request path or public user feature.

## Implementation sequence (future work)

1. Resolve the minimal contract decisions below. Confirm synthetic record shapes
   with maintainer-approved sanitized samples; do not collect personal transcripts.
2. Add one daemon-private module, using `serde_json` and existing test tools.
   Keep provider parsing separate from event construction without introducing
   an adapter framework. No new dependency unless the timestamp decision needs it.
3. Implement bounded record framing before semantic extraction: scan horizon,
   single-record cap, partial-tail handling, byte offsets, discard state and
   event-budget cursor semantics. Never use unbounded `read_to_end`/`lines`.
4. Normalize text/tool blocks, preserve source order, validate supplied session
   identity, and produce structural summaries. Add bounded correlation and
   duplicate-candidate diagnostics; incomplete pairs remain explicit.
5. Add small synthetic fixture files and meaningful parser tests, including
   chunked-reader equivalence and limit boundaries. No environment/home lookup,
   wall clock, random IDs, network or running daemon in these tests.
6. Run focused tests, then appropriate workspace checks; record exact counts,
   commands and limitations. Recheck diff against protected paths and refresh
   Brain status. Stop before runtime/UI/IPC wiring.

## Estimated files

Planning/support material currently consists of three root analysis documents
plus the local Brain scaffold, one contribution page and agent entry points.
Keep support context distinguishable from a potential upstream parser patch;
whether any Brain files belong upstream is an explicit packaging decision for
the user/agy, not an assumption of this plan.

| Future parser patch path | Change / estimate |
|---|---|
| `crates/nebula-daemon/src/session_timeline.rs` | New private implementation and inline unit tests, roughly 300–500 lines together |
| `crates/nebula-daemon/src/lib.rs` | One private `mod session_timeline;` declaration; no runtime caller |
| `crates/nebula-daemon/tests/fixtures/session_timeline/basic.jsonl` | Invented user/assistant text, Bash call/result, edit call/reported result, mixed blocks |
| `crates/nebula-daemon/tests/fixtures/session_timeline/drift.jsonl` | Missing/invalid time, extra fields, unsupported blocks, metadata, conflicting session |
| `crates/nebula-daemon/tests/fixtures/session_timeline/correlation.jsonl` | Multiple/interleaved calls, result-only user envelope, unmatched/duplicate IDs, repeated UUID/message fragments |
| `crates/nebula-daemon/tests/fixtures/session_timeline/privacy.jsonl` | Invented credential/URL/path canaries in every raw content surface and unknown tool names |

Estimate: **6 source/test files**, around 350–600 lines total. Generate huge-line,
UTF-8 split, malformed JSON and incomplete-final-record inputs directly in tests
to keep fixtures small. An expected-output file is optional only if it improves
review clarity. Current planning work creates none of these parser files.

## Deterministic test matrix

| Case | Required assertion |
|---|---|
| Normal text and mixed blocks | Correct source-ordered kinds and source positions; no extra user event for a result-only envelope |
| Interleaved calls/results | Match only exact session/source-scoped IDs; no adjacency/name heuristic; unmatched/ambiguous results remain visible |
| Missing/equal/reversed times | Null plus diagnostics for missing/invalid time; stable physical order; no fabricated clock values |
| Repeat scan / duplicates | Same byte range yields identical output and IDs; identical records at distinct offsets stay distinct; fragments sharing message IDs are not dropped |
| Schema drift | Extra fields tolerated; unknown shapes and malformed complete lines counted; remaining complete records still parsed |
| Incomplete tail | No event until record is complete; cursor does not skip the deferred prefix |
| UTF-8 / chunking | Reader chunk size does not change offsets/events; no panic or byte/character confusion |
| Oversize input / budgets | Record, scan, event and diagnostic bounds respected; deterministic stop reason and resumable record/block cursor; no hidden unbounded dedupe/correlation map |
| Privacy | Invented sensitive canaries absent from summaries/diagnostics; no thinking, arguments, output or absolute paths retained; no terminal controls echoed |
| Session mismatch / read failure | Defined diagnostic/error, no cross-session attribution or raw data in error text |

Tests must exercise the production parser. A fixture-format check without that
parser would only validate sample conventions and is not evidence of parsing
correctness. No standalone test implementation is added in this planning phase.

## Validation commands

Run from the Nebula root. Commands below are planned, **not executed** by this
documentation task. The new-module filter will match tests only after coding;
a zero-test result is not acceptance evidence.

```sh
cargo test -p nebula-daemon --lib session_timeline --locked
cargo test -p nebula-daemon --lib session_context --locked
cargo test -p nebula-daemon --lib session_title --locked
cargo test -p nebula-daemon --lib session_model --locked
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --locked -- -D warnings
cargo build --workspace --locked
cargo test --workspace --locked
```

Later IPC/UI integration would also need `cargo test -p nebula-core --lib
protocol --locked` and `cargo test -p nebula-tui --lib review_modal --locked`,
plus mixed-version and async loading tests. They are not substitutes for parser
tests or for verifying actual History data loading.

## Maintainer decisions and proposed defaults

| Decision | Proposed starting point (not yet approved) |
|---|---|
| Privacy | Structural templates with no content snippets; raw detail deferred |
| Scan/record/event budget | Explicit caller limits; evaluate 8 MiB scan, 1 MiB record, 500 events, 200-character summary as testable starting defaults, not measured guarantees |
| Budget stop inside a multi-block record | Prefer a cursor with next block index; ensure re-reading does not lose or duplicate events |
| Time representation | Nullable validated source RFC3339, preserve offset/precision; choose validation utility/dependency before coding |
| Duplicate handling | Retain distinct physical records and report candidates; no UUID/message-ID collapse until versioned samples justify it |
| Source lifecycle | Caller supplies generation; no file discovery or rotation implementation in pure P0 |
| Subagents/sidechains/compaction | One source only; count unsupported shapes, do not claim complete session coverage |
| History and IPC | Separate follow-up; agree UX, pagination, detail authority and version gates before touching protocol/UI |
| Broader adapters | Codex/Pi later map into the internal contract with independent fixtures; none implemented now |

These uncertainties affect semantics and support claims; they do not block
preserving the research or reviewing this plan. Rebase/revalidate source
references if the eventual contribution starts from a different commit.
