# Architecture

The components of tokentee, how data flows between them, and the decisions that shape them.
**Status: draft.** Each open decision below becomes an ADR in `decisions/` once it is settled.

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

## Open decisions

Each one gets settled in a design session and written up as an ADR.

1. **Runtime and distribution** (ADR-0001, M0)
   - *Python*: fastest for analysis, matches the author's other tooling; the proxy needs an async
     HTTP dependency, and distribution is pipx/uv.
   - *Go*: one static binary holding proxy + store + API + page; best at streaming proxies; analysis is
     more verbose.
   - *TypeScript/Node*: same language as purplemux; good at streaming; heavier to install for a CLI.
   - Leaning: Go for a single binary that stays up as a daemon. The deciding question is whether
     the recorder and API are the product's core (Go) or an add-on to reports (Python).
2. **Storage** (ADR-0003, M1/M2)
   - SQLite in WAL mode (one writer, concurrent readers, one file) is the default candidate.
   - Prompt blocks stored once, keyed by content hash. Consecutive requests share almost all blocks,
     so storage grows with new content, not with request count.
   - Retention: how long bodies are kept versus numbers (numbers forever, bodies N days?).
3. **Block segmentation** (M2): what counts as a block (tool definitions, system parts, each message
   content block) and how `cache_control` breakpoints map onto blocks. This defines what "the block
   that changed" means in diagnosis.
4. **Recorder placement and routing** (M2)
   - How the upstream is configured.
   - Keeping Claude Code in first-party mode behind a proxy (otherwise the 1M context and tool search
     are lost).
   - Behaviour when upstream is down.
5. **Security** (M2/M3)
   - Never store auth headers.
   - Bind to localhost only.
   - Remote access through a tunnel (SSH/Tailscale) rather than an exposed port: an ADR before any
     phone work.
6. **API and live updates** (M3): REST over the store; SSE for live updates vs polling; versioned from
   day one because purplemux depends on it.
7. **Where the UI lives** (M3/M4): the built-in page stays minimal (a fallback and for people without
   purplemux), and rich views live in purplemux? Or does tokentee own a full dashboard?

## Out of scope

- Changing requests (compression, routing, model choice). tokentee observes; the one exception is
  keep-alive, which only re-sends an identical request.
- Hosting for other people. Everything is local and first-party.
