# Architecture

The components of tokentee, how data flows between them, and the decisions that shape them.
**Status: draft.** The decision register below lists what is still open.

## Shape

```
 sources                    core                         consumers
 ───────                    ────                         ─────────
 Claude Code transcripts ─┐
 OpenTelemetry (later) ───┼─► ingest ─► store ◄─ analysis ◄─ report (CLI: text/json/html)
 recorder proxy ──────────┘            (SQLite)    ▲        ◄─ api ◄─ built-in page
                                                   │                ◄─ purplemux tab
                              prices (data) ───────┘
```

One rule holds the design together: **sources write records, consumers read records, nothing reads
a source directly.** Adding a source never touches a report; adding a view never touches a source.

## Components

| Component | Responsibility | Milestone |
|---|---|---|
| `record` | the `Request` record and `Block` types: the one internal contract | M0 |
| `sources/transcripts` | read Claude Code JSONL → records (dedupe per message id, subagent files, compaction) | M1 |
| `store` | persist records and blocks; incremental ingest (remember how far each file was read) | M1 |
| `prices` | per-model rates as a data file with a version; unknown model = tokens only, no dollars | M1 |
| `analysis` | chains, hit rate, miss classification, miss cost, keep-alive what-if. Pure functions | M1 |
| `report` | CLI output: text, JSON, static HTML | M1 |
| `recorder` | streaming proxy; records usage + prompt blocks; forwards to a configured upstream | M2 |
| `diagnosis` | compare consecutive requests' blocks: which block first differs, and why | M2 |
| `api` | HTTP JSON API + live updates over the store | M3 |
| `audit` | pair before/after records of one request across a compressing hop | M5 |
| `keepalive` | expiry scheduler and policy inside the recorder | M6 |

## The record

One row per API request, whatever the source. Key fields: source, `request_id`, session, chain (main
thread or subagent), model, start time, uncached input, cache read, cache write 5m and 1h, output, stop
reason, latency. The recorder adds: ordered list of prompt block hashes, where the `cache_control`
breakpoints are, request and response byte counts.

**Correlation key:** Anthropic's `request-id` response header. Transcripts store it as `requestId`,
OpenTelemetry as `request_id`, and the recorder sees the header. So the same request seen by two
sources merges instead of double-counting. This is why the key is `request_id` and not a local id.

## Decision register

Every decision that shapes tokentee, and **when** it must be settled. `Settled by` is the latest
milestone whose design needs it. `Depends on` shows what has to come first. What can be decided now is
what M0 needs. The rest waits, or is taken `provisional`. Format and statuses:
`references/adr-format.md` in the eudaldrg-workflows core plugin. Most rows will end up `design doc`
rather than ADRs.

| # | Question | Status | Settled by | Depends on | ADR |
|---|---|---|---|---|---|
| 1 | Runtime and distribution: one language or several, installed how? | open | M0 | | |
| 2 | Record schema: which fields, and `request_id` as the join key across sources | open | M0 | 1 | |
| 3 | Change policy for the record schema and the store (no compatibility promise before 1.0? migrations?) | open | M0 | | |
| 4 | Storage engine and layout (SQLite in WAL mode is the candidate) | open | M1 | 1, 3 | |
| 5 | Incremental ingest: how far each transcript was read, and rewritten or compacted files | open | M1 | 4 | |
| 6 | Price catalogue: format, update path, unknown models | open | M1 | | |
| 7 | Block segmentation: what a prompt block is; how `cache_control` breakpoints map onto blocks | open | M2 | 2 | |
| 8 | Recorder placement and routing: upstream config, first-party flags, upstream down | open | M2 | 1 | |
| 9 | Credentials and exposure: never store auth headers, localhost only | open | M2 | | |
| 10 | Body retention: numbers kept forever, bodies for how long? | open | M2 | 4, 7 | |
| 11 | API shape, versioning, and live updates (SSE or polling) | open | M3 | 4 | |
| 12 | Where rich UI lives: built-in page vs purplemux | open | M3 | 11 | |
| 13 | Request pairing across hops (sandwich mode) | open | M5 | 8 | |
| 14 | Keep-alive policy: when to ping, budget, per-model economics | open | M6 | 8 | |
| 15 | Remote access (phone off-LAN): tunnel vs exposed port with auth | open | later | 11 | |

Options already on the table for #1:

- *Python*: fastest for analysis; the proxy needs an async HTTP dependency; installed with pipx or uv.
- *Go*: one static binary for proxy + store + API + page; strongest at streaming proxies; analysis is
  more verbose.
- *TypeScript*: same language as purplemux; heavier to install as a CLI.

The deciding question: are the recorder and the API the core of the product (Go), or an add-on to
reports (Python)?

## Out of scope

- Changing requests (compression, routing, model choice). tokentee observes; the one exception is
  keep-alive, which only re-sends an identical request.
- Hosting for other people. Everything is local and first-party.
