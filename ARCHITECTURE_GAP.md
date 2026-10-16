# Session timeline: architecture gap

Status: research complete; implementation proposal, not an accepted upstream design.
Reviewed on 2026-10-16 (Asia/Taipei), at commit
`a00b76075913bb098021177295e0dab37d3521aa` (workspace version 0.44.0).
Scope: [Issue #144](https://github.com/AgentSystemLabs/nebula/issues/144).
Source locations below refer to that commit, not a moving branch.

## Recommendation

Start with a daemon-private, read-only Claude JSONL parser, exercised through
synthetic fixtures. Accept an explicit reader, session context and limits; return
normalized events and diagnostics. Do not connect it to hooks, a periodic sweep,
IPC, storage or the Review modal in P0. The event proposal is in
[EVENT_SCHEMA.md](EVENT_SCHEMA.md); delivery and validation are in
[P0_PLAN.md](P0_PLAN.md).

The issue ultimately asks for an inspectable history of prompts, tools, commands
and edits, with selection, filtering and historical detail. A transcript parser
provides evidence for that feature; it does not by itself implement replay,
historical diffs or a working History tab.

## What the repository actually does

Read: `README.md`, `ARCHITECTURE.md`, `docs/orchestrator.md`, the Review overlay
and its state/event handling, and the daemon ingestion/context paths.

Nebula is a Rust workspace. The daemon owns PTYs, SQLite and status; the ratatui
TUI communicates over length-prefixed MessagePack on a Unix socket. Harness
hooks separately POST to an authenticated loopback HTTP receiver. Quitting the
TUI leaves agents running. These are existing mechanisms, not timeline storage.

### Claude hook and transcript ingress

```text
spawn -> managed Claude hooks -> authenticated loopback POST
      -> HookPayload -> HookDelivery -> daemon drain
                                   +-> status / recent prompts / cwd handling
                                   +-> remembered TranscriptRef -> title sync

remembered TranscriptRef -> generic session-context read -> flattened text
                        -> model sweep -> current model metadata
```

| Source | Observed responsibility |
|---|---|
| `crates/nebula-daemon/src/registry/spawn.rs:418` and `:559` | Select/install hook dialect; pass Nebula identity and receiver credentials to the child. |
| `crates/nebula-daemon/src/hooks/installer.rs:58` and `:118` | Install event commands and forward stdin JSON with fail-soft curl; timeout is 3 seconds. `PreToolUse` is limited to `AskUserQuestion`, while `PostToolUse` is unmatched. |
| `crates/nebula-daemon/src/hooks/mod.rs:249` | Loopback receiver; delivery channel capacity 256. |
| `crates/nebula-daemon/src/hooks/mod.rs:361` | Authenticate, permissively decode, translate event, and enqueue delivery. Only the Claude route builds `TranscriptRef` from `transcript_path` and `session_id`. |
| `crates/nebula-daemon/src/lib.rs:189` | Drain delivery into status, recent prompts, remembered transcript/title sync, cwd reparenting and pending moves. |
| `crates/nebula-daemon/src/registry.rs:191` | In-memory transcript lookup keyed by Nebula `AgentId`; initialized empty at `:251`. |
| `crates/nebula-daemon/src/session_title.rs:198` | `note_transcript` updates that lookup. |

Nebula agent identity, Claude resumable session identity and subagent identity
are different concepts. A parser must not substitute one for another.
Hook payload/delivery types do not retain source timestamps or `tool_use_id`;
status events are lossy and include synthetic terminal signals. There is no
durable hook journal from which to reconstruct tool start/result history.

### History, context and terminal replay are distinct

| Surface | Actual data path at the reviewed commit |
|---|---|
| Review History | **No fetching or history rendering.** `crates/nebula-tui/src/ui/overlay.rs:50` renders static messages; History at `:81` points users to a CLI command. |
| Review state/open | `crates/nebula-tui/src/review_modal.rs:7` stores session/tab/layout state. `crates/nebula-tui/src/event_loop/server_events.rs:489` opens the overlay on `ReviewOpened`; it does not load history. |
| `nebula orchestrator read` | `crates/nebula-tui/src/ipc.rs:702` -> `crates/nebula-daemon/src/server.rs:363` -> `orchestrator.rs:98` -> `session_context.rs:116`. Role-gated, ANSI-cleaned live PTY tail, capped at 256 KiB, returned as `OutputTail`. No transcript read. |
| `nebula read` | `crates/nebula-tui/src/ipc.rs:521` -> `crates/nebula-daemon/src/server.rs:461` -> `session_context.rs:94`. Prefers remembered Claude transcript, otherwise PTY; returns plain `SessionText`. |
| PTY attach/replay | `crates/nebula-core/src/protocol.rs:57` and `:730`; `crates/nebula-daemon/src/pty/mod.rs:563`. `Attach` replays `Scrollback` then streams `Output`, using byte sequence positions. Ring capacity is 1 MiB (`pty/mod.rs:26`). |

`docs/orchestrator.md` describes History starting with bounded tail output and
Review reusing other surfaces; the current overlay is still a tab shell with
placeholders. Also, the comment at `session_context.rs:100` says orchestrator
read shares transcript logic, but `orchestrator_read` actually calls the PTY
helper. Treat implementation as the evidence; leave upstream docs/UI/comments
unchanged in this analysis iteration.

## Reuse versus unsafe shortcuts

| Existing code | Reuse | Limitation |
|---|---|---|
| `session_title.rs:54`, `TranscriptRef` | Future known-source lookup and Claude session identity | A pathname is not a trust policy or persisted discovery index. |
| `session_context.rs:323`, JSONL/context renderer | Existing shape examples and deterministic tempfile test pattern (`:678`) | Flattens content, discards timestamps/IDs, and can render tool-result text under a User label. Do not change this established text API. |
| `session_title.rs:104`, `transcript_tail` | A bounded-tail precedent | Starts inside a record and omits old history; `read_to_end` after stat is not a strict bound while a file grows. |
| `session_model.rs:58`, `read_appended` | Complete-newline cursor pattern | Reads appended data into an unbounded vector; sweep intentionally skips old history. |
| `prompt_history.rs:62`, `condense` | Control-character/whitespace/clipping conventions | Presentation cleanup is not secret redaction. Recent prompts are a separate, lossy history. |

Smallest proposed code footprint: `crates/nebula-daemon/src/session_timeline.rs`
and one private module declaration beside `session_context/model/title` in
`crates/nebula-daemon/src/lib.rs:16`. Put unit tests beside the private parser.
Do not add a generic adapter framework or move types into `nebula-core` yet.

## Risks and boundaries

| Risk | P0 treatment / later decision |
|---|---|
| Privacy | Summaries use structural templates; omit raw prompts, thinking, commands, paths, tool arguments/results. Log counters, never raw JSON. Opaque source references still need access control before later detail access. |
| Schema drift | Parse only supported record/block shapes; tolerate extra fields, count unknown types, isolate malformed records, defer unfinished last record. Fixtures are synthetic, not a supported-version certification. |
| Duplicate events | Keep physical source identity separate from provider UUID/message ID and tool-call correlation. Retain/report duplicate candidates; do not merge on text, tool name, adjacency or message ID alone. Do not merge hooks in P0. |
| Memory/latency | Bound scan bytes, record bytes, returned events, summary size, diagnostics and correlation state. `BufRead::lines()` alone does not bound a huge line. No parsing under registry/store locks or on an async hook hot path. |
| Discovery/restart | Remembered references are volatile and are not carried as transcript refs in handoff state. Durable Claude session IDs do not imply a discoverable transcript after restart. No home-directory scan in P0. |
| Paths/rotation | Future file opener must validate authority, regular-file policy and allowed roots (including configured Claude home); source generation/cursors must detect replacement and truncation. Pure parser gets an explicit reader. |
| Historical truth | Tool request is intent; result success is reported outcome. Neither guarantees an exact historical file diff or complete command output. Do not derive historical diffs from the current worktree. |
| IPC compatibility | Protocol current/minimum are 50/49, frame cap 4 MiB (`nebula-core/src/protocol.rs:13`, `:20`, `:36`). Internal P0 types do not alter this surface. |

Future IPC changes require a version bump, safe defaults for additive fields and
peer gating for new variants, per `ARCHITECTURE.md`. Compact MessagePack is used
(`crates/nebula-core/src/codec.rs:31`); `Hello`/`HelloOk` are not an additive
extension point. The general broadcaster at `daemon/src/server.rs:1148` does
not filter event variants by peer version. A later timeline event must not be
broadcast indiscriminately to old TUIs or overload `SessionText`/`OutputTail`.

## Open decisions for the maintainer

1. Is History intended to become a transcript timeline, or should bounded PTY
   output remain a separate view?
2. Is a structural, fully content-omitting summary acceptable for the first
   parser? Readable snippets require a separate privacy policy.
3. Should P0 expose pages of bounded source-order events, and what budgets are
   acceptable? What signals make incomplete coverage visible?
4. How should resumed sessions, sidechains and subprocess/subagent files be
   presented? P0 supports one explicitly supplied source only.
5. Which Claude versions/shapes can maintainers validate with sanitized samples?
6. Who owns later discovery, raw-detail authorization and IPC/version gating?

This document records observed behavior and a proposed seam. No maintainer
acceptance, production parser, adapters or UI integration is claimed.
