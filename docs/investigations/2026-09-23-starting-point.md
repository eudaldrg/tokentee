# Starting point (2026-09-23)

What was learned before tokentee had any code: the data sources, a working prototype of the M1 analysis
checked against real usage, and the constraints around proxies. The design sessions start from here.

## Claude Code transcripts as a source

- `~/.claude/projects/<cwd-slug>/<session>.jsonl`, one JSON object per line. Subagents write their own
  files under `<session>/subagents/*.jsonl`.
- An `assistant` line carries `message.usage`. **Claude Code writes one line per content block, and all
  of them repeat the same `message.id` and usage**. Dedupe by `message.id` or requests are counted
  several times over.
- `usage` has `input_tokens` (uncached), `cache_read_input_tokens`, `cache_creation_input_tokens`,
  `cache_creation.{ephemeral_5m_input_tokens, ephemeral_1h_input_tokens}`, `output_tokens`. Both TTLs
  occur in practice, sometimes in one session.
- Each line has `requestId` (Anthropic's `request-id`), `timestamp` (when the response was written, not
  when the request started), `sessionId`, `cwd`, `version`, `model`.
- `model: "<synthetic>"` lines are not API calls. Non-Anthropic model ids appear when a proxy routes
  elsewhere, and they have no Anthropic pricing.
- Missing from transcripts: request bodies, auxiliary calls (title generation, compaction). So a cost
  total from transcripts runs below Claude Code's own tally.

## Claude Code OpenTelemetry as a source

`CLAUDE_CODE_ENABLE_TELEMETRY=1` exports `claude_code.token.usage` (type `input` / `output` /
`cacheRead` / `cacheCreation`), `claude_code.cost.usage`, and with `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`
a `claude_code.llm_request` span (`request_id`, `client_request_id`, `ttft_ms`, cache token counts,
`query_source` = main / subagent / auxiliary). The docs say the `api_request` events are emitted "on
first-party API connections". **Untested: which of these survive a proxy in `ANTHROPIC_BASE_URL`.**

## Prompt-cache economics

- Cache writes cost 1.25× base input (5-minute TTL) or 2× (1-hour TTL); reads refresh the TTL at no
  extra cost. The TTL counts from the start of the request that wrote or read the entry.
- Claude Opus 5.5: $4 / MTok input, $0.20 read. A read is 20× cheaper than uncached input and 40× cheaper
  than a 1-hour write, so a miss costs relatively more than on older models (reads at 0.1×).
- Keep-alive: re-sending the previous request with `max_tokens: 0` refreshes the entry and bills only a
  cache read. Only a component holding the exact request bytes can do it.

## The prototype analysis (M1)

Written against transcripts, outside this repo. The algorithm, to reimplement:

- **Chain** = one transcript file (main thread or one subagent) × one model. The cache is per model,
  and a subagent has its own prefix.
- For each request after the first in its chain: `expected` = previous request's whole prompt
  (`input + read + write5m + write1h`). **Miss** if `expected ≥ 4096` and `read < expected / 2`;
  `rebuilt = expected − read`.
- **Expired** if the gap since the previous request exceeds the TTL in force (1 h if the last write in
  the chain was 1 h, else 5 min), otherwise **invalidated** (the prefix changed while warm).
- **Miss cost** = `rebuilt × (write rate − read rate)`.
- **Keep-alive what-if** for an expired miss: `floor(gap / (TTL − 30 s))` pings × `expected` × read
  rate. Count the saving only where it is cheaper than the miss.
- Known error: gaps are measured between response times, so each is off by one generation time.

**Validation:** per-session cost from this model came out 8–15% below Claude Code's own recorded
session cost (the gap is the auxiliary calls missing from transcripts). That is good enough to compare
misses.

**One real week, as a benchmark shape:** about 4,000 requests; 97.5% of prompt tokens read from cache;
misses about 10% of spend, split roughly 55/45 expired/invalidated. Keep-alive, used only where
cheaper, would have recovered about 40% of the miss cost. Gaps between requests: 94% under 5 min, 2.6%
5–60 min, 0.7% over 1 h. Misses concentrate in a few long sessions with large contexts.

## Proxies

- The standard is the API itself: a proxy accepts `POST /v1/messages` (streamed server-sent events) and
  forwards it. Claude Code finds the first hop through `ANTHROPIC_BASE_URL`.
- A non-Anthropic `ANTHROPIC_BASE_URL` makes Claude Code drop to a 200k context window and turn off
  tool search, unless it is told the upstream is first-party (Claude Code environment flags). The
  recorder must handle this when it is first in the chain.
- caveman, one such proxy, can install itself in "native mode": it writes `ANTHROPIC_BASE_URL` pointing
  at itself into `~/.claude/settings.json`, so every session goes through it, on a fixed port. Putting
  tokentee before it means owning that setting. Putting it after needs caveman's upstream to be
  configurable (probably; unconfirmed).
- caveman already records a per-request row in its own SQLite database, including per-component prompt
  prefix hashes and compression before/after, plus the original text of compressed items. That makes an
  optional adapter possible, built only from the database's observable schema. Its source is partly
  BSL-1.1 licensed: never read or copy it here.

## The purplemux consumer

purplemux is Next.js/TypeScript, charts with recharts, and already has a stats feature
(`src/components/features/stats/`) and a mobile panel. The natural split: tokentee serves an API, and
purplemux renders the rich views, desktop and phone.
