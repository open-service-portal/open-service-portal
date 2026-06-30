# Specs & knowledge layout

This workspace follows the **spec-driven TDD** method (the team's `craft:spec-driven-tdd`). Work starts from a spec, the PR opens at step 1, tests encode the acceptance criteria, and spec/tests/code/docs stay in sync.

## Layout (PD doctrine)

| Location | Purpose |
|---|---|
| `specs/<feature>/` | **One folder per feature.** Product specs as `NNNN_product_<feature>_<topic>.md` (numbered; a feature may have several). Optional `tech-spec` lives here or in the implementing repo. |
| `docs/decisions/` | Dated architecture/design decision records (ADR-style). |
| `plans/<feature>.md` | Multi-step implementation plans — scaffolding, dropped when the work lands. |
| `journal/` | One short entry per PR (`YYYY-MM-DD-<slug>.md`, flat). |
| `docs/` | Current-state reference documentation (explains the components). |

**`specs/` is never under `docs/`.** Spec = intent ("what & why"); docs = explanation of the components. Mixing them loses both (PD principle). `specs/` is top-level.

## Spec anatomy (fixed hierarchy)

```
spec (one theme)
└── user story  (US-<feature>-N)        top-level item
    └── acceptance criterion            never top-level
        (AC-<feature>-N-M + test ref: Unit | Integration | E2E)
```

Header line (not a table): `` Spec-ID: `SPEC-<feature>` · Status · Datum · Autor ``. Numbered sections.

## Flow (per feature)

Spec first (open the PR now) → tests for each AC → plan in `plans/` (if multi-step) → implement until spec/tests/code/docs agree → journal entry → **merge on approval only**.

## osp deviations from PD (documented on purpose)

osp is a **public** platform repo, not the internal plugin-marketplace repo PD's conventions grew in. Two conscious deviations:

1. **Language: English.** PD specs are German (internal); osp is public → all committed content is English.
2. *(under review)* Whether specs should also be **published** on the docs site (`docs.openportal.dev`). PD keeps specs out of the published surface (intent ≠ docs); osp currently does too (`specs/` is not synced to the site). If we ever want them visible, the Docusaurus sync can include `specs/` explicitly.

These deviations are tracked back to PD to evolve the shared method (see the `knowledge-architecture` kit).
