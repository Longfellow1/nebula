# Proposed internal timeline event contract

Status: draft for Issue #144; no Rust types or IPC changes implemented.
Evidence baseline: `a00b76075913bb098021177295e0dab37d3521aa`, 2026-10-16.
See [ARCHITECTURE_GAP.md](ARCHITECTURE_GAP.md) for source paths and
[P0_PLAN.md](P0_PLAN.md) for implementation/test scope.

## Minimal event

This is an internal semantic contract, not a new serialized wire protocol.

| Field | Proposed meaning |
|---|---|
| `id` | Deterministic tuple of source generation, record start byte offset and block index (a distinct record-level sentinel for string content). Stable across appends to that generation. |
| `timestamp` | Nullable source time, validated RFC3339 with explicit offset; preserve its original precision/offset in P0. Missing/invalid -> null plus diagnostic. Never substitute read time, file mtime or ULID time. |
| `kind` | `user_message`, `assistant_message`, `tool_call`, `tool_result`. Unknown provider types do not create arbitrary new kinds. |
| `session` | `{ nebula_agent_id, harness, harness_session_id }`, supplied explicitly by the caller; P0 harness is `claude`. |
| `tool_call_id` | Nullable provider call ID: `tool_use.id` on calls, `tool_result.tool_use_id` on results. Scope correlation to the same session and source generation. |
| `source` | `{ source_id, generation, byte_offset, block_index, record_uuid? }`. Source ID is opaque; no absolute path. Offset is zero-based UTF-8 byte position; array block index is zero-based. |
| `redacted_summary` | Bounded, control-free, structural summary from an allowlist; never a raw-content excerpt. |

Optional small metadata: `tool_kind` = `command | file_edit | other` for tool
calls/results, and result `outcome` = `reported_success | reported_error |
unknown`. Classify only recognized tool names, never shell-parse arbitrary
commands. These are filters, not additional command/edit events: one source
tool request must not produce duplicate generic and specialized events.
Carry classification from a matched call only when available; unmatched results
remain valid with unknown classification. Maintainer may defer this metadata.

The harness session ID differs from Nebula `AgentId`. If a record explicitly
declares a conflicting session ID, reject that record with a count/diagnostic;
if absent, use the supplied context without inventing a provider identity.
Subagent streams are outside P0 rather than silently merged into the parent.

## Normalization rules

1. Read complete newline-terminated JSON records in physical source order;
   preserve array block order. Do not sort by timestamp: timestamps may be
   missing, equal or out of order. Multiple blocks inherit the envelope time.
2. Recognize `message.role` and string/array `message.content` for supported
   Claude `user`/`assistant` records. Existing fallback text forms in
   `session_context.rs` are evidence to investigate, not automatically included
   in the first supported grammar.
3. Inspect block type before outer role. `tool_use` -> call; `tool_result` ->
   result, even inside a user envelope. A result-only user record emits no
   additional user-message event. Mixed text and tool blocks emit separate
   events at their original block positions.
4. Ignore title/model/attachment/progress/summary/thinking/image records or
   blocks unless a shape is deliberately added later; count unsupported shapes.
   Do not present a compacted summary as a complete recovered conversation.
5. A syntactically valid record with an unsupported shape is different from
   malformed JSON. Missing call IDs do not get guessed from tool names or
   neighboring records. Report missing/unmatched/ambiguous correlation.
6. An unfinished final record is deferred, with the cursor left at its start.
   A malformed complete record can be skipped and counted. Oversize records
   require bounded discard-to-newline; retain enough cursor state to avoid
   reinterpreting a suffix as a new record.

Tool blocks follow known Claude tool-message semantics; persisted JSONL is a
separate, version-dependent format. The repository has title/model fixtures
with version notes, not a comprehensive versioned transcript specification.
Support claims require maintainer-confirmed sanitized transcript samples.

## Example (synthetic; not a captured transcript)

Input uses invented identities/content and newline-terminated records:

```jsonl
{"type":"user","uuid":"r1","sessionId":"claude-demo","timestamp":"2026-10-16T00:00:00Z","message":{"role":"user","content":"Synthetic task text"}}
{"type":"assistant","uuid":"r2","sessionId":"claude-demo","timestamp":"2026-10-16T00:00:01Z","message":{"role":"assistant","content":[{"type":"tool_use","id":"call-demo","name":"Bash","input":{"command":"printf synthetic"}}]}}
{"type":"user","uuid":"r3","sessionId":"claude-demo","timestamp":"2026-10-16T00:00:02Z","message":{"role":"user","content":[{"type":"tool_result","tool_use_id":"call-demo","content":"synthetic","is_error":false}]}}
```

Expected sequence: `user_message`, `tool_call`, `tool_result`. The last two
share `tool_call_id=call-demo`; the first has null correlation. Example call
event below uses a symbolic byte offset `N` (not literal output JSON):

```text
id: (source-demo, generation-1, N, 0)
timestamp: 2026-10-16T00:00:01Z
kind: tool_call
session: {nebula_agent_id: nebula-demo, harness: claude, harness_session_id: claude-demo}
tool_call_id: call-demo
source: {source_id: source-demo, generation: generation-1, byte_offset: N, block_index: 0, record_uuid: r2}
tool_kind: command
redacted_summary: Command requested [arguments omitted]
```

## Privacy policy for P0

Use templates such as `User message [content omitted]`, `Assistant message
[content omitted]`, `Command requested [arguments omitted]`, `File edit
requested [arguments omitted]`, and `Tool result [content omitted]`.
Map unknown tool names to a generic template, rather than echoing arbitrary
provider strings. Omit thinking, prompt/response text, commands, filenames,
URLs, arguments, environment values and output bodies from normalized objects.
Do not serialize or log the raw JSON tree. All retained strings, including IDs,
need type/length/control checks; caller-owned IDs remain potentially identifying
metadata and are not an anonymized export format.

Length clipping, ANSI stripping and `prompt_history::condense` do not redact
secrets. Even opaque source IDs and offsets confer no authority to read raw
detail. P0 never resolves them to user-visible content; later detail access
needs independent session/source authorization and a chosen disclosure policy.

## Identity, duplicates and coverage

Physical event identity, provider UUID/message identity and tool-call
correlation are separate. Repeated equal text can be a legitimate new event;
message IDs can be shared by fragments. Retain physical duplicates and report
candidate repeated UUIDs/call IDs within bounded state; do not silently collapse
them. Re-reading the same source range reproduces the same IDs, enabling a
future caller to discard repeated physical events. Rewrites/truncation/replacement
invalidate the source generation and any old cursor; a pure parser does not
infer filesystem generations. No hook/transcript fusion in P0.

Proposed parse result: `{ events, diagnostics, coverage }`. Coverage contains
start/next record byte offsets, scanned bytes, a bounded scan horizon, whether
the horizon was fully processed, deferred-tail state and explicit stop reasons
(`byte_budget`, `event_budget`, etc.). Diagnostics use bounded counters and
positions/codes, not raw source text. Unknown records, parse errors, oversize
records and unresolved tool results make omissions visible even when scanning
reaches the horizon. Never describe bounded coverage as a complete session.

A future file reader should snapshot a byte horizon and use strict read limits
so concurrent appends do not expand a request indefinitely. Correlation may be
incomplete across pages; never fabricate a match. If continuation stops within
a multi-block record, the cursor must include the next block or the parser must
stop before that record atomically. P0 design/tests must choose one rule.

## Codex / Pi extension seam

Keep provider grammar behind a small normalization function. Later Codex and Pi
adapters can accept their own explicit source/session context and emit these
events, null source times and provider correlation IDs where available. They
must supply their own fixtures, schema/version policy and identity mapping.
Claude-shaped hooks do not establish transcript compatibility. Do not implement
adapters, a registry, file discovery, cross-provider merging or shared IPC types
in this iteration.

Unresolved before coding: accepted Claude grammar/versions; timestamp-validation
dependency versus existing utilities; default budgets; physical-duplicate policy;
cursor behavior at block boundaries; privacy templates and tool classification.
