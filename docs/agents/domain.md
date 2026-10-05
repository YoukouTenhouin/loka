# Domain docs

This repo uses a single-context layout:

- `CONTEXT.md` at the repo root: domain vocabulary and model.
- `docs/adr/`: architectural decision records.

## Before exploring

Read `CONTEXT.md` and ADRs relevant to the area being explored.

If these files do not exist, proceed silently. Domain modeling
creates them when terms or decisions are resolved.

## Use the glossary's vocabulary

Use the terms defined in `CONTEXT.md` in issue titles, proposals,
hypotheses, tests, and other outputs.

If a concept is missing, reconsider whether the project needs it.
Record genuine vocabulary gaps for domain modeling.

## Flag ADR conflicts

If a proposal contradicts an existing ADR, identify the ADR and
explain why its decision should be reconsidered.
