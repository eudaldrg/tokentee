# Roadmap

Milestones from nothing to a useful public tool, each with a goal, an exit test and the design work it
needs first. **Status: draft**, not yet reviewed. The first review pass decides the MVP cut and the order.

Each milestone gets one design doc in `docs/design/` before any implementation plan is written. Each
decision it forces gets an ADR in `decisions/`. GitHub issues track milestones and link here; this file
and the design docs are the source of truth.

## M0: Foundations

- **Goal:** a repo someone else could clone, build and test.
- **Scope:** language/runtime, layout, CI (lint + tests), a commit hook against personal data, synthetic
  fixtures, and the `Request` record schema v0 (see architecture).
- **Exit:** `make test` (or equivalent) is green in CI on a fixture-only codebase.
- **Design needed:** ADR-0001 runtime and distribution, ADR-0002 record schema.

## M1: Transcript report (proposed MVP)

- **Goal:** zero-setup answer to "how well is my prompt cache used, and what do misses cost?"
- **Scope:** Claude Code transcript source → store → analysis (request chains, hit rate, misses split
  into expired and invalidated, miss cost, what keep-alive would have cost instead) → report as text,
  JSON and a static HTML page. Price catalogue as data.
- **Exit:** `tokentee report --since 7d` on a real machine gives numbers within about 15% of Claude
  Code's own cost tally, with fixtures covering every miss kind.
- **Why it is the MVP:** it needs nothing but files Claude Code already writes, so anyone can try it
  in a minute, and it is enough to decide whether the rest is worth it.

## M2: Recorder

- **Goal:** see *why* a miss happened.
- **Scope:** a passthrough proxy for the Anthropic Messages API (streaming, unbuffered) that writes one
  record per request plus the prompt split into content-addressed blocks. Miss diagnosis: which block
  first differs from the previous request in the chain. Chaining before or after another proxy.
- **Exit:** a deliberately edited system prompt shows up as "invalidated at block N (system)", and the
  added latency is under 5 ms to the first byte.
- **Design needed:** block segmentation, storage and retention, placement and routing, credential
  handling.

## M3: Live view and API

- **Goal:** watch sessions as they happen, from any screen.
- **Scope:** a local HTTP API over the store (sessions, requests, misses, block diffs) with live
  updates, plus a minimal built-in page. The API is the contract other screens use.
- **Exit:** a miss appears in the page within 2 s of the response finishing.
- **Design needed:** API shape and versioning, push mechanism (SSE vs polling), what the built-in page
  covers versus purplemux.

## M4: purplemux tab

- **Goal:** the numbers where you already look, desktop and phone.
- **Scope:** a tab next to purplemux's stats, reading the M3 API. Lives in purplemux, not here.
- **Design needed:** which views, how purplemux discovers the API.

## M5: Compression audit

- **Goal:** know whether compression in the chain pays for itself and never over-compresses.
- **Scope:** recorder on both sides of a compressing proxy ("sandwich"), pairing the two records of one
  request; per-block before/after, tokens saved versus cache effects, and a browser for original versus
  compressed text.
- **Design needed:** request pairing across hops, the savings accounting (a compression that busts the
  cache can cost more than it saves).

## M6: Cache keep-alive

- **Goal:** stop paying for expiries you could have prevented cheaply.
- **Scope:** the recorder re-sends the last request with `max_tokens: 0` just before the cache expires,
  under a policy (only while a session looks active, budget cap, per-model economics); savings
  accounting; a status line signal.
- **Design needed:** policy, what "session is idle but coming back" means, failure modes.

## Later / backlog

- OpenTelemetry ingest as a third source.
- Remote access (phone outside the LAN): tunnel versus exposing with auth.
- Other providers' APIs (OpenAI Responses, Gemini).
- Alerts and budgets; cross-machine aggregation.
