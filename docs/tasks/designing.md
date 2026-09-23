---
aliases: [design, design-session, adr]
sources: [docs/roadmap.md, docs/architecture.md, docs/design/**, decisions/**]
---

# Designing

How design work is done and resumed in this repo.

Design runs through the `/core:design` skill (eudaldrg-workflows core plugin). It owns the process:
roadmap, then architecture and its decision register, then one design doc per milestone, then
`/core:simplify-design` before a milestone is `ready` for `/core:generate-plan`. ADRs follow its
`references/adr-format.md`: one expensive-to-reverse decision each, `proposed` by default.

Where things live here:

- `docs/roadmap.md`: milestones, MVP, design status
- `docs/architecture.md`: components, data flow, **decision register**
- `docs/design/mN-<slug>.md`: one per milestone
- `decisions/NNNN-<slug>.md`: ADRs; superseded ones in `decisions/superseded/`
- `docs/investigations/`: evidence, starting with `2026-09-23-starting-point.md`

A fresh session reads, in order: this file, the roadmap, the architecture, the starting-point
investigation, then the design doc of the milestone at hand. Nothing about the design lives only in a
conversation.
