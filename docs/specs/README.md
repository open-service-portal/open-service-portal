# Specs & knowledge layout

This workspace follows a spec-driven convention (aligned with the team's craft method):

| Location | Purpose |
|---|---|
| `docs/specs/<feature>/` | **One folder per feature.** `product-spec.md` (mandatory) = product intent; `tech-spec.md` (optional) = technical cut, or lives in the implementing repo. |
| `docs/decisions/` | Dated architecture/design decision records (ADR-style; formerly `concepts/`). |
| `plans/<feature>.md` | Multi-step implementation plans — scaffolding, dropped when the work lands. |
| `journal/` | One short entry per PR (session, result, learnings). |
| `docs/` (rest) | Current-state reference documentation. |

## Spec anatomy (fixed hierarchy)

```
spec (theme)
└── user story  (US-NNN)            top-level item
    └── acceptance criterion        never top-level
        (AC-NNN-N + test ref: Unit | Integration | E2E)
```

- A spec covers **one theme**, written as **multiple user stories**, each with its own ID.
- Under each user story hang **multiple acceptance criteria**, each with its own ID and a **test reference**.
- If an AC is genuinely not testable (informational/manual), say so explicitly on the AC.

## Flow (per feature)

Spec first (open the PR now) → tests for each AC → plan in `plans/` (if multi-step) → implement until spec/tests/code/docs agree → journal entry → merge **on approval only**.
