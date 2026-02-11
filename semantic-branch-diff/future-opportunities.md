# Config 2.0 Semantic Diff — Future Opportunities

**Origin**: [824295 — CM Import: Reliable Diff Elimination](https://o9git.visualstudio.com/CoreDev/_workitems/edit/824295)

*Extensions of the CM 3.0 architecture — not in scope for initial delivery.*

---

## The Core Insight

CM 3.0's semantic config pipeline introduces primitives that are **Config 2.0 platform capabilities**, not CM-specific constructs. CM 3.0 is the first consumer; it won't be the last.

Two reusable building blocks:

1. **Compiled Solution Snapshot** (`IComparisonContext`): A solution's complete entity state, deserialized into strongly typed RPCModels. Can be produced from a Git branch, a loaded tenant, or a config backup.

2. **Semantic Change Tree** (`ComparisonNode`): A tree of entity-level changes with source/target objects, change types, and the full entity hierarchy. Produced by comparing any two snapshots.

These two primitives generalize far beyond branch-vs-branch diff. Any Config 2.0 workflow that needs to understand what changed, validate state, or compare environments can build on them.

---

## Opportunity 1: Any-to-Any Comparison

The `CompareConfigWorkflow` takes any two `IComparisonContext` instances. With the Git-based implementation, three data sources exist:

| Source | Target | Use Case |
|--------|--------|----------|
| Solution branch | Solution branch | Semantic branch diff (CM 3.0) |
| Solution branch | Runtime tenant | Drift detection: what's different between Git and what's loaded? |
| Config backup | Solution branch | Backup validation |
| Runtime tenant | Runtime tenant | Cross-tenant comparison |
| Config backup | Runtime tenant | Already supported |

Each combination is just a different pair of `IComparisonContext` implementations — no new comparison logic needed. The compare engine doesn't know or care where the data came from.

## Opportunity 2: Solution Validation

Compiling a solution snapshot into an `IComparisonContext` is itself a validation step. If deserialization fails, the solution state is broken. This can be exposed as a standalone Config 2.0 operation:

- **Pre-deployment check**: Compile a branch before loading — does it deserialize cleanly?
- **Completeness audit**: Verify all expected entity types are present and well-formed
- **Schema compatibility**: Detect entities that won't deserialize on the current platform version
- **Referential integrity**: Validate cross-entity relationships after compilation

No comparison needed — just compile one snapshot and inspect it. This is useful for any C2 workflow that touches solution state: promotion, loading, CM import, backup restore.

## Opportunity 3: ComparisonNode as a Universal Config Change Format

The `ComparisonNode` tree carries: entity type (`RpcType`), change type (Added/Modified/Removed), and both source and target deserialized objects. This makes it a bidirectional interchange format for the entire Config 2.0 platform:

```
Git file changes ←→ ComparisonNode tree ←→ CRUD operations (TenantDB)
                          ↕
                   IComparisonContext (snapshots)
```

### Git → ComparisonNode (reverse of current GitSync adapters)

The ~60 `IGitAdapter` implementations already encode the mapping between Git file paths and entity types (e.g., `Dimensions/{name}/Dimension.json` → `Dimension`). Reversing this mapping:

- Take a Git commit's changed files
- Use path structure to identify entity type
- Deserialize file content into RPCModels
- Construct `ComparisonNode` with source/target objects

This gives a **semantic understanding of a Git commit** — not "these files changed" but "these entities changed in these ways." Any C2 feature that consumes Git changes (promotion, CM packaging, sync) benefits from this.

### ComparisonNode → CRUD (applying a semantic diff to a tenant)

Each `ComparisonNode` has everything needed to dispatch a CRUD operation:

- `RpcType` → which CRUD controller
- `ChangeType` → POST (Added), PUT (Modified), DELETE (Removed)
- `TargetObject` → the entity payload

This is essentially a **config DB patcher** driven by semantic diffs. Potential applications across Config 2.0:

- **CM package import**: Currently Git diff → package (a JSON artifact with entity contents, change info, and dependency info) → then separately, package → CRUD import to TenantDB → Git sync. Could become: semantic diff (ComparisonNode) → package → CRUD, where the package is derived from semantic understanding rather than Git file diffs — guaranteeing diff elimination
- **Incremental config restore**: Instead of full DB replace, apply only the differences
- **Selective sync**: Pick specific `ComparisonNode` subtrees to apply
- **Config migration**: Apply a diff from one environment to another

## Opportunity 4: Tenant as a Snapshot Source (Version Resolution)

A solution branch implies a uniform branch name across all config blocks. But a loaded tenant uses whatever config block versions are actually loaded — which may differ per block. Both resolve to the same thing: a complete `IComparisonContext`.

This means the semantic diff API doesn't need to be limited to branch names. The parameters identify **how to resolve a snapshot**, and a tenant is one valid resolution strategy alongside branch names. This generalizes the API beyond CM 3.0's branch-vs-branch use case to any Config 2.0 scenario that needs to compare or inspect state.

---

*These opportunities are natural extensions of the CM 3.0 architecture. They don't require separate infrastructure — they reuse the same `IComparisonContext`, `ComparisonNode`, and deserialization primitives.*

---

*See also:*
- *[Requirements](requirements.md) — CM 3.0 scope, dependency chain, and what it solves*
- *[Analysis](analysis.md) — Codebase investigation and approach*
- *[Architecture Notes](architecture.md) — Implementation detail*
- *[CM 2.0 Problem Landscape](../../c2-requirements/config-requirements/cm2-strategic-enhancements-pov.md) — Full problem inventory*
