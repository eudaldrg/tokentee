# tokentee

Agent instructions for this repo. `CLAUDE.md` is a symlink to this file; edit `AGENTS.md`.

## What this is

A public, MIT-licensed tool for LLM usage statistics (see `README.md`). One record format per API
request; one adapter per data source (Claude Code transcripts, Claude Code OpenTelemetry, the tokentee
recorder proxy). Reports and dashboards read only the record format, never a source directly.

## Hard rules

- **Public repo, no personal data.** Never commit transcripts, recorded requests, databases, real
  session ids, project names from the author's machine, or anything else from real usage. Fixtures are
  synthetic.
- **No code from caveman.** It is partly BSL-1.1 licensed. Don't copy, port, or read its source while
  implementing equivalents here. Referring to it by name ("works alongside caveman") is fine.
- **The recorder never stores credentials** (`authorization`, `x-api-key`, cookies, any auth header),
  and never buffers a streamed response before relaying it.

## Docs

`docs/modules/` holds one doc per component, `docs/tasks/` one per procedure (the eudaldrg-workflows
knowledge convention).
