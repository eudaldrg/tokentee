---
aliases: [design, design-session, adr]
sources: [docs/roadmap.md, docs/architecture.md, docs/design/**, decisions/**]
---

# Designing

How design work is done and resumed in this repo, so a fresh session can pick up exactly where the
last one stopped.

## The three levels

1. **Roadmap** (`docs/roadmap.md`): milestones, MVP cut, exit tests. Each milestone has a GitHub issue
   linking back here.
2. **Architecture** (`docs/architecture.md`): components, data flow, contracts, and the list of open
   decisions.
3. **Milestone design** (`docs/design/mN-<slug>.md`): for one milestone, detailed enough that an
   implementation plan (`/core:generate-plan`) can be written from it without re-deciding anything.

Decisions are **ADRs** in `decisions/NNNN-<slug>.md`: context, options with pros and cons, decision,
consequences. Status `proposed` while discussed, `accepted` when settled. Design docs cite ADRs rather
than repeating them.

## Resuming

A new session starts by reading, in order: this file, `docs/roadmap.md`, `docs/architecture.md`
(especially **Open decisions**), then the design doc for the milestone being worked on. Nothing
about the design lives only in a conversation.

## Rules

- Design first, implementation later: no milestone gets an implementation plan until its design doc
  has no open questions that change the plan.
- Give options with pros and cons and a recommendation; the author decides.
- When a decision changes, supersede the ADR (new ADR, old one marked `superseded by`). Never
  rewrite history.
