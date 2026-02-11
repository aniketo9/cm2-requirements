# CM 3.0 — Semantic Config Pipeline

**Origin**: [824295 — CM Import: Reliable Diff Elimination](https://o9git.visualstudio.com/CoreDev/_workitems/edit/824295)

Started as a diff elimination fix for CM 2.0. Codebase analysis revealed the diff can't be decoupled from the package pipeline — and that rebuilding both solves 13 of 15 known CM 2.0 problems. That makes it CM 3.0.

## Documents

| Document | What it covers |
|----------|---------------|
| [principles.md](principles.md) | Core design principle: one authoritative code path per data flow direction |
| [requirements.md](requirements.md) | Starting requirement, dependency chain, what it solves, success criteria |
| [analysis.md](analysis.md) | Codebase investigation: existing infrastructure, feasibility, approach comparison |
| [architecture.md](architecture.md) | Implementation notes for semantic diff (Steps 1-3) and full pipeline redesign (new package model, import workflow — Steps 4-6) |
| [future-opportunities.md](future-opportunities.md) | Platform-level extensions beyond CM 3.0: any-to-any comparison, solution validation, ComparisonNode as universal change format |
| [evolution.md](evolution.md) | How the thinking evolved across sessions — from one-paragraph requirement to full pipeline redesign |

## Repository

These requirements are maintained at [github.com/aniketo9/cm2-requirements](https://github.com/aniketo9/cm2-requirements).
