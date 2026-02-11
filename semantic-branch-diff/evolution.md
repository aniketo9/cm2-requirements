# Semantic Branch Diff — Evolution of Ideas

**ADO Work Item**: [824295 — CM Import: Reliable Diff Elimination](https://o9git.visualstudio.com/CoreDev/_workitems/edit/824295)

This document captures how the thinking evolved across sessions. Each insight built on the previous one — the final architecture is significantly different from where we started.

---

## Session 1: From one-paragraph requirement to deep codebase analysis

**Starting point**: A one-paragraph requirement from ADO 824295 — "diff elimination doesn't work, make the diff semantic."

**Key moves**:

1. **Explored the existing infrastructure.** Mapped the full existing stack: the Git diff API (`gitdiffs`), the compare engine (`CompareConfigWorkflow`), the `IComparisonContext` abstraction (with two implementations: `GlobalDbBasedComparisonContext` for loaded tenants, `BackupBasedComparisonContext` for backup files), and the Compiler pipeline (18 compiler classes that deserialize Git content into RPCModels and persist to TenantDB).

2. **Discovered the key invariant.** The original requirement was implicitly per-file. Codebase analysis revealed the comparison unit must be the **entire solution** — entities span config blocks, and only full deserialization produces complete objects. The invariant: *two solution versions that produce the same entity state after deserialization must compare as identical, regardless of how the underlying Git files look, how many files are involved, or how they are structured.*

3. **Collapsed the effort estimate.** Analysis of the 18 compilers showed DB IDs (the apparent coupling between deserialization and persistence) are invisible to the compare engine. This collapsed the estimate from ~16-20 weeks to ~5-8 weeks (manual) or ~1.5-2 weeks (AI-assisted).

4. **Identified the solution snapshot concept.** A branch name resolves to a complete solution snapshot (solution block + all config blocks, compiled). A loaded tenant is also a snapshot. The diff becomes: compare any two snapshots. This led to the `IComparisonContext` abstraction as the core primitive.

5. **Surfaced Config 2.0 framing.** The primitives being built (compiled snapshots, semantic change trees) aren't CM 2.0-specific — they're platform capabilities. CM 2.0 is the first consumer, not the only one. Reframed the future-opportunities doc around Config 2.0.

**Output**: [requirements.md](requirements.md) (problem, invariant, scope, success criteria), [analysis.md](analysis.md) (codebase investigation, feasibility, approach comparison), [architecture.md](architecture.md) (API design, compiler inventory, code sketches), [future-opportunities.md](future-opportunities.md) (any-to-any comparison, solution validation, ComparisonNode as universal change format).

---

## Session 2: From diff-only to full pipeline redesign

**Starting point**: The diff architecture was solid. But: what happens after the diff? The diff feeds into package creation and import — and that's where things broke down.

**Key moves**:

6. **Discovered block-scoped package constraint.** The existing `PublishedModelPackage` has a single `ModuleId` — it's tied to one config block. The semantic diff produces a solution-level tree. These don't fit together.

7. **Discovered derived Git entities.** Some entities in Git adapters aren't first-class package entities — `WorkspaceGitAdapter` has position properties, `DefaultPageName` is a computed field. The `IsOnlyPositionChanged()` pattern suppresses position-only changes. These exist only in the Git layer and have no package representation.

8. **Discovered `GetPublishedConfigEntities` redundancy.** This API re-reads Git, re-deserializes, and queries TenantDB for parent DB IDs by name — all work the semantic diff already completes. The `PublishedConfigEntity.ParentEntityId` is a DB record ID (`int`), not a GUID. The ComparisonNode tree already carries everything this API produces and more.

9. **Realized fundamental pipeline incompatibility.** The current pipeline's killer feature: it can process **any arbitrary subset of changed files** independently, because parentage comes from live DB queries. The semantic diff requires full solution compilation before any comparison. You can't do one with the other. The question became: can we adapt the old pipeline? The answer was no.

10. **Decision: new solution-level packages.** The existing package pipeline was built on CM 1.0 assumptions (file-level diffs, DB-dependent parentage, block-scoped packages) and has not stabilized despite years of bug-fixing. Incremental adaptation would compound the problem. Decision: build entirely new solution-level packages with a custom import workflow that natively consumes the ComparisonNode tree.

11. **Designed the new package model.** Sketched `ConfigChangePackage` and `ConfigChangePackageEntity` — solution-scoped, GUID-based identity, tree-based parentage, embedded entity payloads. No `ParentEntityId`, no `ModuleId` at the package level.

12. **Tackled import ordering.** Entity interdependencies require CRUD operations in the correct order (Dimensions before Plans, Plans before Measures). Investigated where this ordering is defined in the codebase.

13. **Discovered three independent representations of entity ordering.** The `EntityTree` (deeply nested, used by backup/restore — walks top-down for creates, bottom-up for deletes), the Compiler sequence (hard-coded sequential `await` calls in `Compiler.CompileAsync()`, coordinated through shared `EntityNameIdCache`), and the `CompareEntityTree` (flat, used for comparison — no cross-entity ordering). All three agree on direction (Locales -> Dimensions -> Plans -> Measures -> Rules -> UI) but are maintained independently.

14. **Identified EntityTree as the reusable ordering source.** `TenantConfigStructure.EntityTree` is `public static readonly`, production-proven for both top-down restore and bottom-up delete, and each handler has `RpcType` that maps to entity types. The `RestoreWorkflow.DoRestore()` and `RestoreWorkflow.DoDelete()` traversal patterns are exactly what the import workflow needs. The dependency ordering should NOT go into RPCModels (which are DTOs) — the EntityTree is the right place and it already exists.

15. **Flagged the two-tree gap.** The comparison uses `CompareEntityTree` (flat structure), but import ordering uses `EntityTree` (deeply nested). These need to be bridged: given a ComparisonNode from the flat comparison tree, determine its position in the EntityTree's ordering for CRUD dispatch.

16. **Found the simple bridge: `RpcType` is the join key.** Both `ComparisonNode.RpcType` and `IEntityHandler.RpcType` are `System.Type`. The dispatch algorithm is: flatten the ComparisonNode tree into a `Dictionary<Type, List<ComparisonNode>>`, then walk the EntityTree depth-first — at each handler, look up its RpcType in the dictionary and dispatch. Structurally identical to `RestoreWorkflow.DoRestore()`/`DoDelete()`.

17. **Discovered online vs offline CRUD distinction.** The platform has two sets of CRUD controllers per entity type: online (`api/onlinecrud/...`, DDL through live LS) and offline (`api/crud/...`, direct DB, requires LS startup after). The choice is driven by the **destructive nature of changes** — destructive changes must use offline CRUD because LS can't process DDL that removes schema elements it's using. `PackageExecutorBase.ForceOfflineCrud` encodes this today. For the new import, `ChangeType` on each ComparisonNode naturally determines the mode.

---

## Session 3: Document consolidation, design principles, and import problem coverage

**Starting point**: The four documents existed but had significant overlap. Analysis, architecture, and requirements covered some of the same ground without clear boundaries. The problem landscape had 12 items; the architecture addressed 8.

**Key moves**:

18. **Reviewed analysis.md for correctness.** Fixed code examples against the actual codebase: `rpcAtt.DimensionId` → `rpcAttTrans.DimensionAttributeId` in the entityNameIdCache example, `LoadModulesAsync()` → `CompileAsync()` in the Files Reference. Removed forward-looking content (Any-to-Any Comparison, Compiled Snapshot as a Building Block) that belonged in future-opportunities.md. Replaced "What's Wrong With This Pipeline" with a cross-reference to requirements — the analysis was duplicating the problem statement.

19. **Reviewed architecture.md for net-new value.** Identified what was net-new (API design, proposed flow, implementation steps, compiler inventory, entity coverage) vs. pure duplication. Removed: "Why This Works" table (5 rows restating analysis findings) → one paragraph + cross-reference. Step 4 pipeline incompatibility explanation (~30 lines) → cross-reference to requirements. Risks trimmed from 9 rows to 3 implementation-specific rows. Files Reference trimmed to architecture-specific additions. Fixed broken anchor (`#why-this-requires-a-new-pipeline` → `#from-diff-fix-to-pipeline-redesign`) and `LoadModulesAsync()` → `CompileAsync()`.

20. **Recognized more problems as addressed.** User identified that #6 (Performance) and #14 (APIs Not Automation-Friendly) are both directly solved: performance problems stem from anti-patterns in create and import (re-reading Git, re-deserializing, DB parentage queries), and API problems stem from multi-call UI orchestration. The new pipeline eliminates all of this — package creation consumes the ComparisonNode tree directly.

21. **Simplified the API to branch-vs-branch.** The initial design overloaded query parameters to support branch-vs-branch, branch-vs-tenant, and tenant-vs-tenant in one endpoint. Simplified to: `GET /api/config/semanticdiff?solutionName={solution}&fromBranch={source}&toBranch={target}`. Tenant comparison uses the same engine but gets its own API surface later.

22. **Identified TenantDB dependency elimination.** The old pipeline required a running TenantDB for package creation (`GetPublishedConfigEntities` queries the target tenant for parent DB IDs). The new pipeline eliminates this for phases 1–4: diff, entity selection, and package creation are pure Git-to-Git operations. Only phase 5 (import) requires a running tenant.

23. **Clarified tenant as version resolution, not a data source.** When using a tenant as one side of the comparison, the tenant isn't a different data source — it resolves to specific Git versions (each config block has a loaded branch/tag), and deserialization proceeds from Git as usual via the Compiler. Same path, different version resolver.

24. **Articulated the core design principle.** Created [principles.md](principles.md): one authoritative code path per data flow direction. Git→TenantDB = Compiler. TenantDB→Git = GitSync. Git→ComparisonContext = new CompilerBasedComparisonContext. ComparisonNode→TenantDB = package import via CRUD. User→TenantDB = CRUD APIs. CM 2.0's problems trace to violating this — `GetPublishedConfigEntities` was a second deserialization path that diverged from the Compiler.

25. **Expanded the problem landscape from 12 to 15.** Three new import problems added to the [CM 2.0 Problem Landscape](../../c2-requirements/config-requirements/cm2-strategic-enhancements-pov.md): #8 (import doesn't respect entity dependencies), #9 (child entity import is destructive rather than differential), #10 (import failures can be silent). All three are addressed by the new architecture — EntityTree provides dependency ordering, per-entity ComparisonNodes make import differential, and discrete CRUD operations make per-entity tracking natural. **The pipeline now addresses 13 of 15 known problems** (up from 8 at end of Session 2). Only #3 (GUID Lifecycle) and #5 (Atomicity) remain outside scope.

**Output**: [principles.md](principles.md) (new), updated [requirements.md](requirements.md) (TenantDB independence, tenant as version resolver, 13/15 problem coverage), updated [architecture.md](architecture.md) (simplified API, cross-references, removed duplication), updated [analysis.md](analysis.md) (trimmed, cross-references).

---

## Open Questions

- Atomicity: should imports be all-or-nothing? (POV Problem #5)
- Git sync timing: batch after all CRUD vs. per-entity?
- Conflict detection: compare target tenant's current state against package's expected snapshot before import
- Package serialization format for `SourceObject`/`TargetObject`
- Online/offline CRUD strategy: always offline + LS restart for simplicity, or partition into online/offline batches based on `ChangeType`?
- Whether to extract a single authoritative dependency graph that all consumers (backup/restore, compiler, comparison, import) derive from, instead of maintaining four independent representations

---

*See also:*
- *[Principles](principles.md) — One authoritative path per data flow*
- *[Requirements](requirements.md) — Problem statement, success criteria*
- *[Analysis](analysis.md) — Codebase investigation and approach comparison*
- *[Architecture Notes](architecture.md) — Implementation detail*
- *[Future Opportunities](future-opportunities.md) — Extensions beyond 824295*
