# tokentee

Usage statistics for LLM coding sessions: prompt-cache hits and misses, what each miss cost and why
it happened, and what any compression in the chain actually saved.

Like Unix `tee`, the recorder passes the stream through untouched and writes a copy to the side.

> **Status: early.** Nothing to install yet; this README describes where it is going.

## Data sources

tokentee works from one record per API request, and reads that record from whichever source you have:

| Source | Setup | What it gives |
|---|---|---|
| Claude Code transcripts (`~/.claude/projects/*.jsonl`) | none | per-response usage, cache TTL split, timing: hit rate and miss cost |
| Claude Code OpenTelemetry | `CLAUDE_CODE_ENABLE_TELEMETRY=1` | the same, plus request ids and main/subagent/auxiliary attribution |
| the tokentee recorder | point `ANTHROPIC_BASE_URL` at it | request contents: which prompt block changed on a miss, before/after compression |

## The recorder in a chain

The recorder speaks the Anthropic Messages API on both sides, so it can sit anywhere in a chain of
proxies without them knowing it is there:

```
Claude Code ─► [other proxy] ─► tokentee ─► api.anthropic.com
```

After another proxy, it sees the exact bytes the provider receives, which are what cache hits depend on.
Before another proxy, it sees what the client really sent. In both positions it compares before and after.

Guarantees it has to keep: responses stream through unbuffered, and credentials and auth headers are
never stored.

## Origin

tokentee started as a way to audit a local [caveman](https://github.com/JuliusBrussee/caveman) proxy
setup. It shares no code with caveman and does not depend on it. Any proxy that speaks the Messages API
can sit next to it.

## License

MIT
