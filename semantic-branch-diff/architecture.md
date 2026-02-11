# CM 3.0 Semantic Config Pipeline — Architecture Notes

**Origin**: [824295 — CM Import: Reliable Diff Elimination](https://o9git.visualstudio.com/CoreDev/_workitems/edit/824295)

---

## Scope Evolution

This document started as architecture notes for 824295 (semantic branch-vs-branch diff). During requirements analysis, it became clear that the diff cannot be decoupled from the downstream package and import pipeline — they are one system. Replacing the diff while keeping the old pipeline creates an impedance mismatch (see [requirements.md — From Diff Fix to Pipeline Redesign](requirements.md#from-diff-fix-to-pipeline-redesign)).

The document now covers two phases:

| Phase                          | Steps     | What it covers                                                                                                          |
| ------------------------------ | --------- | ----------------------------------------------------------------------------------------------------------------------- |
| **Phase 1: Semantic Diff**     | Steps 1–3 | Read solution branch content, deserialize via Compiler, run CompareConfigWorkflow → `ComparisonNode` tree               |
| **Phase 2: Pipeline Redesign** | Steps 4–6 | New solution-level package model, `ConfigChangePackage` object, dependency-ordered import workflow with online/offline CRUD |

**These phases are not independently shippable.** The semantic diff (Phase 1) produces a `ComparisonNode` tree, but the existing package pipeline cannot consume it — it requires the old Git file diff as input (see [requirements.md — Why the Existing Pipeline Cannot Be Reused](requirements.md#why-the-existing-pipeline-cannot-be-reused)). A semantic diff without a new package pipeline doesn't solve diff elimination. Both phases must be delivered together for the problem to be solved.

---

## New API Endpoint

### Endpoint

**Current**: `GET /api/config/git/gitdiffs?moduleId={blockId}&fromBranch={source}&toBranch={target}`

- Scope: Single config block
- Returns: `List<GitEntityCommitInfo>` (Git file-level changes)

**New**: `GET /api/config/semanticdiff?solutionName={solution}&fromBranch={source}&toBranch={target}`

- Scope: Entire solution (all config blocks, compiled as a single unit)
- Returns: `ComparisonNode` tree (semantic entity-level changes)
- No `moduleId` parameter — operates at solution level
- **No loaded tenant required**: The solution name identifies the solution block's Git repo, which defines which config blocks exist — so config block discovery comes from Git, not from a loaded tenant. This makes it a pure Git-to-Git operation that can run before a solution is loaded, from a different tenant, or from a standalone service.

A tenant can also be used as one side of the comparison — not as a different data source, but as a version resolution shortcut. The tenant resolves to the specific Git branches/tags loaded per config block, and deserialization proceeds from Git as usual via the same Compiler path. See [Requirements — Tenant as a Version Resolution Shortcut](requirements.md#tenant-as-a-version-resolution-shortcut).

### Proposed Flow

```
  Source Snapshot                                       Target Snapshot
  (solution branch OR tenant)                          (solution branch OR tenant)
  ┌─────────────────────────────────────┐              ┌─────────────────────────────────────┐
  │ Resolve config block versions       │              │ Resolve config block versions       │
  │ (branch: uniform branch name;      │              │ (branch: uniform branch name;      │
  │  tenant: per-block loaded versions) │              │  tenant: per-block loaded versions) │
  └──────────────────┬──────────────────┘              └──────────────────┬──────────────────┘
                     │                                                    │
                     ▼                                                    ▼
         ┌───────────────────────┐                        ┌───────────────────────┐
         │ Compile to entity state│                        │ Compile to entity state│
         │ (GitReader.MergeModules│                        │ (GitReader.MergeModules│
         │  + Compiler deserialize)│                       │  + Compiler deserialize)│
         └───────────┬───────────┘                        └───────────┬───────────┘
                     │                                                │
                     ▼                                                ▼
         ┌───────────────────────┐                        ┌───────────────────────┐
         │ IComparisonContext     │                        │ IComparisonContext     │
         │ (source — full solution│                        │ (target — full solution│
         │  compiled state)       │                        │  compiled state)       │
         └───────────┬───────────┘                        └───────────┬───────────┘
                     │                                                │
                     └──────────────┬─────────────────────────────────┘
                                    ▼
                         ┌──────────────────────┐
                         │ CompareConfigWorkflow │
                         │ (TenantCompareHandler │
                         │  → full entity tree)  │
                         └──────────┬───────────┘
                                    ▼
                         ┌──────────────────────┐
                         │ ComparisonNode tree   │
                         │ (semantic diff,       │
                         │  solution-level)      │
                         └──────────────────────┘
```

---

## Implementation Steps

### Step 1: Read Solution Branch Content

For each solution branch, read and assemble the complete Git content. Internally, a solution branch spans the solution block repo and all config block repos — but this step treats them as a single compilation unit.

Use the **solution name** to locate the solution block's Git repo. The solution block itself defines which config blocks exist (module definitions) — so config block discovery comes from Git, not from a loaded tenant. No `loadedModules` call is needed and no loaded tenant is required.

For each repo (solution block + all config blocks), download the branch content via the existing Git API client (ZIP/archive or tree endpoint). The result is the raw material for a single compilation pass.

### Step 2: Deserialize Using the Compiler (Deserialization Only)

Reuse the Compiler's deserialization pipeline — the same code that `LoadModulesAsync()` uses to parse Git JSON into RPCModels. The difference: instead of writing to TenantDB via CRUD, collect the RPCModels into an in-memory `IComparisonContext`.

#### The Extraction Pattern

For each compiler, the pattern is:

```csharp
// CURRENT (CompileAsync — interleaved deserialization + persistence):
foreach (var content in dimensions)
{
    rpcDim = GetSpecialRpcFromJObject<Dimension>(content.EntityJObject);  // ← KEEP
    rpcDim.Id = 0;                                                        // ← skip
    rpcDim.TenantId = TenantId;                                           // ← skip
    var postDimResponse = await dimensionCrud.PostDimensionAsync(...);     // ← skip
    EntityNameIdCache.AddDimensionId(...);                                 // ← skip
}

// NEW (diff path — deserialization only):
foreach (var content in dimensions)
{
    rpcDim = GetSpecialRpcFromJObject<Dimension>(content.EntityJObject);  // same call
    comparisonContext.Add(rpcDim);                                         // collect
}
```

#### Compiler Safety: Parallel Path, No Modifications to Existing Compiler

The existing `CompileAsync()` pipeline is production-critical — it runs on every solution load. The diff path must not modify it. Analysis of `CompilerBase` confirms a **parallel path** is clean and low-risk:

**What's directly reusable (zero duplication — same methods):**

| Method                                      | Location             | Signature                                                                    | Coupling                                               |
| ------------------------------------------- | -------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------ |
| `GetSpecialRpcFromJObject<T>()`             | `CompilerBase`       | **`public static`** — delegates to `Util.GetSpecialRpcFromJObject<T>()`      | **None** — pure JSON → RPCModel, no instance state     |
| `Util.GetSpecialRpcFromJObject<T>()`        | `Git/Common/Util.cs` | Static utility: `JToken.ToObject<T>(serializer)` with `CustomGuidConverter`  | **None** — standalone                                  |
| `GetStableNewGuid()`                        | `CompilerBase`       | Instance method, but delegates to static `O9ConfigGuidUtils.GuidGenerator()` | **None** — can call utility directly                   |
| `IConfig2Reader.GetEntitiesFromFolder<T>()` | `IConfig2Reader`     | Interface method — reads Git content, returns `List<(T Entity, Module)>`     | **None** — data source interface, already decoupled    |
| `GitReader.MergeModules()`                  | `GitReader`          | Module merging before compilation                                            | **None** — operates on Git content                     |
| `CompareConfigWorkflow` + handlers          | `o9.ConfigCompare`   | Compare engine                                                               | **None** — already designed for pluggable data sources |

**What needs a thin wrapper (protected instance methods, simple logic):**

| Method                        | Why not directly callable                     | Mitigation                                                                                           |
| ----------------------------- | --------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `GetBackedUpElementArray()`   | `protected` instance method on `CompilerBase` | Simple logic (parse JArray, filter deleted items) — extract to static utility or duplicate ~25 lines |
| `GetBackedUpElementJObject()` | Same                                          | Same pattern, ~15 lines                                                                              |

**What's duplicated (the "wiring" — ~100-150 lines total across all 18 compilers):**

The foreach iteration logic that connects Git content to deserialization calls. Per entity type it's ~1-2 lines:

```csharp
// This is what's "duplicated" — the iteration + method call:
foreach (var content in config2Reader.GetEntitiesFromFolder<GEDimension>("Dimensions"))
    context.Add(Util.GetSpecialRpcFromJObject<Dimension>(content.Entity));
```

**What's NOT duplicated (the bulk — ~80% of compiler LOC):**

- All persistence code (CRUD calls, `EntityNameIdCache`, `TenantId`/`Id` assignment)
- The deserialization method implementations themselves
- The compare engine and all compare handlers
- Git reading and module merging

**Drift risk and mitigation:** If someone adds a new entity type to a compiler's `CompileAsync()`, they need to also add it to the diff path. A cross-reference test (entity types in diff path vs. `CompareEntityTree` handlers vs. compiler entity types) catches this automatically.

#### Why This Works: DB IDs Are Invisible to the Compare Engine

DB IDs, `EntityNameIdCache`, foreign keys, and sequential write phases are all persistence concerns — none are inputs to deserialization, and the compare engine excludes them. Detailed evidence (50+ call sites verified) is in the [Analysis — Deserialization Is Independent of DB Persistence](analysis.md#deserialization-is-independent-of-db-persistence).

#### New `IComparisonContext` Implementation

```csharp
public class CompilerBasedComparisonContext : IComparisonContext
{
    private readonly Dictionary<Type, IList> _entities = new();

    public void Add<T>(T entity) { ... }
    public void AddRange<T>(IEnumerable<T> entities) { ... }

    public IList<T> ListEntities<T>()
    {
        return _entities.TryGetValue(typeof(T), out var list)
            ? (IList<T>)list
            : new List<T>();
    }

    // GetTenant(), GetLocales(), GetGlobalPlugins() delegate to ListEntities<T>()
}
```

This follows the same pattern as `BackupBasedComparisonContext` — it implements `IComparisonContext` and serves entities from an in-memory collection instead of from disk files or DB.

### Step 3: Run CompareConfigWorkflow

```csharp
var workflow = new CompareConfigWorkflow(
    sourceContext,   // IComparisonContext for source branch (full solution)
    targetContext,   // IComparisonContext for target branch (full solution)
    new CompareConfigOptions
    {
        Strategy = ComparisonStrategy.EntityGuid,
        ResetExcludedProperties = true
    });

var comparisonNode = await workflow.Compare();

// Filter out unchanged nodes
TenantConfigDiffTransformer.FilterOutUnchangedNodes(comparisonNode, null);
```

### Step 4: New Solution-Level Package Model

The existing package pipeline cannot be adapted to consume the semantic diff — the incompatibilities (scoping, parentage model, entity granularity) and the design principles for the new model are covered in the [Requirements — From Diff Fix to Pipeline Redesign](requirements.md#from-diff-fix-to-pipeline-redesign). The current pipeline's `GetPublishedConfigEntities` step (re-reading from Git, re-deserializing, DB parentage lookups) becomes entirely redundant — the `ComparisonNode` tree already carries everything it produces and more.

#### Entity Granularity in the New Model

The compare engine's entity granularity and the current package model's entity granularity differ:

1. **Derived properties in Git**: GitSync adapters apply special handling not reflected in the package model. For example, `WorkspaceGitAdapter`, `DimensionGitAdapter`, `PageGitAdapter`, `PageGroupGitAdapter`, and `ViewGitAdapter` all check `IsOnlyPositionChanged()` — position changes are suppressed from Git sync entirely. `WorkspaceGitAdapter.GetGitEntityFileContent()` computes `DefaultPageName` from child nodes — a derived property that only exists in Git.

2. **Aggregation differences** (POV Problem #9): CM 2.0 packages at the parent entity level (e.g., the measure), while the compare engine sees translations, aliases, and properties as separate ComparisonNode children.

The new package model has the opportunity to **fix this misalignment**: use the CompareEntityTree's entity granularity as the authoritative definition, and let users select at whatever level the tree exposes. This resolves POV Problem #9 (Entity Definition Misalignment) as a side effect.

#### GitSync Adapters Reference

The ~60 `IGitAdapter` implementations in `GitSync/Adapters/` encode the mapping between RPCModel types and Git file paths (entity state → Git). While they go in the opposite direction (not directly usable for deserialization), their **entity type registry and path knowledge** may be useful for:

- The custom import workflow (mapping ComparisonNode entities to CRUD controllers)
- Audit/validation (cross-referencing entity types in the diff path vs. Git sync path)

### Step 5: New Package Object

The current package objects (`PublishedModelPackage`, `PublishedConfigEntity`, `ImportedModelPackage`) are tied to the old pipeline. The new package is a different structure.

#### Current Package Objects (for reference)

| Object                  | Key Fields                                                                                                                            | Coupling                                                    |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `PublishedModelPackage` | `ModuleId` (single block), `PackageJson` (serialized blob), `IsConfigMgmtPackage`, `IsSolutionPackage`, `SourceBranch`/`TargetBranch` | Block-scoped via `ModuleId`                                 |
| `PublishedConfigEntity` | `EntityType` (string), `EntityId` (GUID), `EntityName`, `ParentEntityId` (**DB record ID — int**), `MappingAction`                    | Flat list, parentage via DB ID                              |
| `ImportedModelPackage`  | `PackageId`, `Status`, `IsConfigMgmtPackage`                                                                                          | No block reference, but consumes block-scoped `PackageJson` |

#### New Package Object (sketch)

The new package is a **serialized ComparisonNode subtree** — the user's selection from the full semantic diff, carrying everything needed for import without DB lookups or Git re-reads.

```csharp
public class ConfigChangePackage
{
    public Guid PackageId { get; set; }
    public string SolutionName { get; set; }
    public string SourceBranch { get; set; }
    public string TargetBranch { get; set; }

    // The selected changes — a ComparisonNode subtree
    // Each node carries: RpcType, ChangeType, EntityGuid,
    // SourceObject, TargetObject, Children (tree structure = parentage)
    public List<ConfigChangePackageEntity> Entities { get; set; }

    // Metadata
    public DateTime CreatedAt { get; set; }
    public string CreatedBy { get; set; }
    public string Description { get; set; }
    public bool ContainsDestructiveChanges { get; set; }  // auto-detected from ChangeType
}

public class ConfigChangePackageEntity
{
    public Guid EntityGuid { get; set; }
    public string EntityName { get; set; }
    public string EntityType { get; set; }          // RpcType name
    public string ChangeType { get; set; }          // Added, Modified, Removed
    public Guid ModuleId { get; set; }              // config block attribution (internal)
    public object SourceObject { get; set; }        // deserialized RPCModel (before)
    public object TargetObject { get; set; }        // deserialized RPCModel (after)
    public List<ConfigChangePackageEntity> Children { get; set; }  // tree parentage
}
```

Key differences from the current model:

- **No `ParentEntityId` (DB record ID)**. Parentage is the tree structure itself (`Children`).
- **No `ModuleId` at the package level**. Solution-scoped. `ModuleId` lives on each entity (config block attribution).
- **Embedded entity payloads**. Source and target objects travel with the package — no re-reading from Git or DB at import time.
- **`ContainsDestructiveChanges` auto-detected**. Any entity with `ChangeType = Removed` (or specific destructive modification patterns) flags the package automatically — solving POV Problem #4.

### Step 6: New Import Workflow

The import workflow applies a `ConfigChangePackage` to a target tenant. The core problem is **sequencing**: entity interdependencies require CRUD operations to execute in the correct order (e.g., a Dimension must exist before its Attributes can be created, a Plan before its Measures).

#### Dependency-Ordered CRUD Dispatch

**Important distinction**: The `CompareEntityTree` (used for comparison) is **flat** — entity types like Plan, Dimension, RuleGroup, and Hierarchy are top-level siblings. It only encodes intra-subtree containment (Dimension → DimensionAttribute → DimensionAttributeTranslation), **not** cross-entity-type dependencies (e.g., Measure references Dimension).

The cross-entity dependency ordering is encoded in the separate **`EntityTree`** (`TenantConfigStructure.EntityTree`), which is a deeply nested tree used by backup/restore. In the EntityTree, Dimension is nested above Plan, Plan above MeasureGroup, MeasureGroup above Measure — the nesting depth implicitly encodes that Dimensions must exist before Plans, Plans before MeasureGroups, etc.

**The EntityTree's ordering is directly reusable.** It is production-proven for exactly the two traversal patterns the import workflow needs:

- **`RestoreWorkflow.DoRestore()`** walks the tree **top-down** (parent first, then children) — this is the add/modify ordering
- **`RestoreWorkflow.DoDelete()`** walks the tree **bottom-up** (children first, then parent) — this is the delete ordering
- **`BackupWorkflow.DoBackup()`**, **`ModuleQueryExtensions.DoDeleteModule()`**, and **`TenantConfigBackupValidator`** all use the same tree

Each handler in the EntityTree has a `RpcType` property that maps to the entity type string in `ConfigChangePackageEntity`. A depth-first walk of the EntityTree produces a flat, dependency-ordered sequence of entity types. The import workflow maps each `ConfigChangePackageEntity.EntityType` to this sequence to determine CRUD execution order.

For **additions and modifications** (creates/updates), follow EntityTree order **top-down** (parents first):

```
Dimension → DimensionAttribute → AttributeProperty → ...
Plan → MeasureGroup → Measure → MeasureTranslation → ...
```

For **removals** (deletes), follow EntityTree order **bottom-up** (children first):

```
... → MeasureTranslation → Measure → MeasureGroup → Plan
```

Each node dispatches a CRUD operation:

- `RpcType` → which CRUD controller
- `ChangeType` → POST (Added), PUT (Modified), DELETE (Removed)
- `TargetObject` → the entity payload (for adds/modifications)
- `EntityGuid` → the entity identifier (for deletes)

#### Bridging CompareEntityTree → EntityTree (the dispatch algorithm)

`RpcType` (`System.Type`) is the shared key between both trees. The `ComparisonNode` from the comparison has `RpcType`, and each `IEntityHandler` in the EntityTree also has `RpcType`. The bridge is a dictionary lookup:

```csharp
// 1. Flatten ComparisonNode tree into a lookup by RpcType
var changes = new Dictionary<Type, List<ComparisonNode>>();
FlattenComparisonTree(comparisonRoot, changes);

// 2. Walk EntityTree — ordering falls out naturally
void DispatchCrud(IEntityHandler handler, Dictionary<Type, List<ComparisonNode>> changes)
{
    var nodes = changes.GetValueOrDefault(handler.RpcType, empty);

    // Adds/mods: parent first (top-down) — same as RestoreWorkflow.DoRestore()
    foreach (var node in nodes.Where(n => n.ChangeType != Removed))
        CallCrudApi(handler.RpcType, node);

    // Recurse into children (follows EntityTree ordering)
    foreach (var child in handler.Children)
        DispatchCrud(child, changes);

    // Deletes: children first (bottom-up) — same as RestoreWorkflow.DoDelete()
    foreach (var node in nodes.Where(n => n.ChangeType == Removed))
        CallCrudApi(handler.RpcType, node);
}

// Start from EntityTree root
DispatchCrud(TenantConfigStructure.EntityTree, changes);
```

This is structurally identical to `RestoreWorkflow.DoRestore()` / `DoDelete()` — just replacing "read from backup file and write to DB" with "look up ComparisonNode and call CRUD API."

#### Online vs Offline CRUD Controllers

The platform has **two sets of CRUD controllers** for each entity type:

| Mode        | Route                               | What it does                                                                | When to use                                     |
| ----------- | ----------------------------------- | --------------------------------------------------------------------------- | ----------------------------------------------- |
| **Online**  | `api/onlinecrud/tenants/{tenantId}` | DB operations + DDL commands through Live Server (`ExecuteDdlCommandAsync`) | Non-destructive changes — LS stays running      |
| **Offline** | `api/crud/tenants/{tenantId}`       | Direct DB operations, bypasses Live Server                                  | Destructive changes — requires LS startup after |

This is a **package-level** classification — the entire package is one or the other:

- **Destructive package** (contains any deletes or schema-breaking modifications): Must use **offline CRUD** because the Live Server can't process DDL that removes schema elements it's actively using. After import, an LS startup is required to reload with the new schema.
- **Non-destructive package** (only adds and non-schema-breaking modifications): Can use **online CRUD** — the Live Server processes DDL in-flight (e.g., adding a new measure column), no restart needed.

**Today: manual classification.** The user decides whether a package is destructive during package creation. The flag `IsContainsNonDestructiveChanges` on `PublishedModelPackage` is set by the user and flows through the pipeline:

```csharp
// PackageExecutorBase.cs:110
public bool ForceOfflineCrud => EntityMap.IsConfigManagementPackage && !EntityMap.IsContainsNonDestructiveChanges;
```

Before a destructive import, the system checks for unsaved LS transactions (`IsLsSaveNeeded` — LS must save before destructive changes are applied). Failed offline operations are tracked in `FailedOfflineCrudCommand` for retry.

**New workflow: automatic classification.** The `ComparisonNode` in the newer `o9.ConfigCompare` framework already carries a `HasDataImpactingChanges` flag. This flag is **not** simply derived from `ChangeType` — it's more nuanced:

Each entity compare handler defines which **specific properties** are data-impacting via `GetDataImpactingProperties()`. For example:

- **DimensionCompareHandler**: `DimensionName`, `DimensionTableName`, `DimensionKeyColumnName`
- **MeasureCompareHandler**: `MeasureName`, `MeasureGroupName`, `DataType`, `MeasureColumnName`, `IsReportingMeasure`

During comparison, `ComparisonHelper.AreObjectsEqual()` checks whether any changed property is in the data-impacting list. If so, `HasDataImpactingChanges = true` on that `ComparisonNode`. The flag aggregates up the tree — if any child node has data-impacting changes, the parent does too.

The package-level classification is then:

- If **any** `ComparisonNode` in the package has `HasDataImpactingChanges = true` → destructive package → offline CRUD → LS startup after
- Otherwise → non-destructive package → online CRUD → LS stays running

This replaces the manual user classification with a precise, property-level determination — auto-computed from the compare tree (POV Problem #4). The `ContainsDestructiveChanges` field on the `ConfigChangePackage` sketch is derived directly from the tree's `HasDataImpactingChanges` flags.

After all CRUD operations succeed → trigger Git sync for affected config blocks (identified by `ModuleId` on the entities).

#### Comparison with Current Import

| Aspect                         | Current Import                                                               | New Import                                                                                                           |
| ------------------------------ | ---------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Input**                      | `PackageJson` (serialized blob, block-scoped)                                | `ConfigChangePackage` (tree with payloads, solution-scoped)                                                              |
| **Entity resolution**          | Re-reads from package JSON, resolves by name + DB                            | Already deserialized, identified by GUID                                                                             |
| **Dependency ordering**        | Implicit in import code, fragile                                             | Explicit in tree structure, derived from EntityTree (production-proven in backup/restore)                            |
| **CRUD dispatch**              | Custom per-entity-type import logic                                          | Generic: `RpcType` → controller, `ChangeType` → verb, `TargetObject` → payload                                       |
| **Destructive classification** | Manual — user sets `IsContainsNonDestructiveChanges` during package creation | Auto-detected: `HasDataImpactingChanges` on ComparisonNode, computed per-property via `GetDataImpactingProperties()` |
| **Online/offline CRUD**        | Package-level flag (`ForceOfflineCrud`) based on manual classification       | Package-level: any `HasDataImpactingChanges` in tree → offline + LS startup; otherwise → online                      |
| **Git sync**                   | Per-entity sync during import                                                | Batch sync after all CRUD operations succeed                                                                         |
| **Scope**                      | Block-scoped (single `ModuleId`)                                             | Solution-scoped (entities carry `ModuleId` internally)                                                               |
| **Atomicity**                  | Not atomic — partial failure leaves mixed state                              | Opportunity for transactional batching per block                                                                     |

#### Open Questions for the Architect

1. **Atomicity**: Should the import be all-or-nothing? The tree structure makes it possible to group CRUD operations into a transaction per config block. Current imports are not atomic (POV Problem #5).
2. **Git sync timing**: Sync after all CRUD operations (batch) vs. sync per entity? Batch is simpler and avoids intermediate Git states.
3. **Conflict detection**: Before import, compare the target tenant's current state against the package's expected target snapshot. Divergence = conflict. This is an any-to-any comparison (Opportunity 1 in [future-opportunities.md](future-opportunities.md)).
4. **Package serialization**: `SourceObject`/`TargetObject` are RPCModels. Serialization format (JSON with type discriminator? Protobuf?) affects package size and portability.
5. **Incremental application**: Can a package be partially applied (user selects a subset at import time, not just at creation time)?
6. **Two trees**: The `CompareEntityTree` (used for comparison) and the `EntityTree` (used for import ordering) are separate structures with different topologies. The ComparisonNode tree follows CompareEntityTree's flat structure (Plan and Dimension are siblings). The import ordering follows EntityTree's deeply nested structure (Dimension before Plan). The dispatch algorithm above bridges these via `RpcType` lookup — validate that all `RpcType` values in CompareEntityTree exist in EntityTree.
7. **Online/offline CRUD strategy**: The current import uses `ForceOfflineCrud` (destructive → offline + LS restart). Should the new import always use offline CRUD for simplicity (at the cost of LS restart), or partition into online/offline batches for zero-downtime non-destructive imports? The per-entity `ChangeType` makes this partition possible automatically.

---

## Compiler Inventory (18 active)

| Compiler                  | LOC  | Deserialization Method          | Entity Types | Extraction Complexity        |
| ------------------------- | ---- | ------------------------------- | ------------ | ---------------------------- |
| LocalesCompiler           | 42   | `GetEntitiesFromFolder<Locale>` | 1            | Trivial                      |
| RolesCompiler             | 77   | `GetEntitiesFromFolder<Role>`   | 1            | Trivial                      |
| RepoDetailsCompiler       | 43   | `GetBackedUpElementArray`       | 1            | Trivial                      |
| LsSequenceCompiler        | 55   | `GetSpecialRpcFromJObject`      | 1            | Trivial                      |
| UiPreferencesCompiler     | 52   | `GetEntitiesFromFolder`         | 1            | Trivial                      |
| FileAccessCompiler        | 75   | `GetEntitiesFromFolder`         | 1            | Trivial                      |
| MassEditTemplatesCompiler | 54   | `GetEntitiesFromFolder`         | 1            | Trivial                      |
| StagingApisCompiler       | 159  | `GetSpecialRpcFromJObject`      | 1            | Trivial                      |
| PickListsCompiler         | 96   | `GetBackedUpElementArray`       | 2            | Simple (nested)              |
| PluginsCompiler           | 292  | `GetSpecialRpcFromJObject`      | 4            | Simple (nested)              |
| EtlConfigCompiler         | 117  | `GetSpecialRpcFromJObject`      | 3            | Simple                       |
| AlertCompiler             | 178  | `GetSpecialRpcFromJObject`      | 3            | Simple                       |
| PlayCompiler              | 110  | `GetSpecialRpcFromJObject`      | 4            | Simple                       |
| CollabCompiler            | 203  | `GetSpecialRpcFromJObject`      | 3            | Simple                       |
| RulesCompiler             | 814  | `GetBackedUpElementArray`       | 5+           | Moderate (type dispatch)     |
| GraphCompiler             | 435  | `GetSpecialRpcFromJObject`      | 8+           | Moderate (nesting)           |
| DimensionCompiler         | 1029 | `GetSpecialRpcFromJObject`      | 12+          | Moderate (many entity types) |
| PlansCompiler             | 1246 | `GetSpecialRpcFromJObject`      | 10+          | Moderate (many entity types) |
| UiConfigCompiler          | 1697 | `GetBackedUpElementArray`       | 15+          | Moderate (many entity types) |

The "moderate" compilers (Dimension, Plans, UiConfig) were originally assessed as "Very Complex" when considering full deserialization/persistence separation. They downgrade to "Moderate" because the complexity was in the 9-10 sequential DB write phases and 20+ `EntityNameIdCache` usages — which are all skipped for the diff path. The deserialization calls themselves are straightforward in every compiler.

---

## Entity Type Coverage

All entity types in `CompareConfigWorkflow.CompareEntityTree` must be collected during deserialization:

| Category          | Entity Types                                                                                                                                                                                           |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Core Model**    | Plans, MeasureGroups, Measures (+ Aliases, Aggregates, Translations, Twins, Spreads, Keywords, Properties, Static Properties, Conditional Formula Properties, MeasureToMeasure Relationships)          |
| **Dimensions**    | Dimensions, DimensionAttributes, Hierarchies, Levels (+ Translations, Aliases, External Configs, Keywords, NamedNodeUsages, MemberSpecificProperties, SummaryMeasures)                                 |
| **Relationships** | MemberRelationshipTypes, NodeAttributeElements, NodeAliases, Properties (+ Translations, External Configs, Static Properties, Conditional Formulas)                                                    |
| **Rules**         | RuleGroupLabels, RuleGroupScopeLabels, RuleGroups, InverseRuleGroupSets, InverseRuleGroups, InverseRules, IbplRules (+ Translations)                                                                   |
| **UI**            | Workspaces, PageGroups, Pages, Views, WidgetDefinitions, XLFolders, XLWorkbooks, XLWidgetInWorkbooks, ActionButtons, ActionButtonBindings, UiPreferences, MassEditTemplates (+ Translations, Keywords) |
| **Alerts**        | Alerts, AlertViewsBindings, Plays, PlayScenarios, DataAlertPlays (+ Translations)                                                                                                                      |
| **Plugins**       | GlobalPlugins, ConfiguredGlobalPlugins, TenantPlugins, ConfiguredTenantPlugins                                                                                                                         |
| **Other**         | Locales, Modules, Picklists, LSSequences, Roles, StagingDataApiInfos, ETLPackages, MemberRelNodeProperties                                                                                             |

Cross-reference this against the `CompareEntityTree` handler list during implementation to ensure no entity type is missed.

---

## Comparison Configuration

```csharp
new CompareConfigOptions
{
    Strategy = ComparisonStrategy.EntityGuid,    // match by GUID, not name
    ResetExcludedProperties = true               // nullify excluded props for clean display
}
```

### Properties Already Excluded by ComparisonHelper

- `Id` (database-generated)
- `TenantId`
- Properties marked `[ExcludeFromCompare]`

### Additional Exclusions to Consider

- `ModuleId` — can differ across branches (`CompareConfigOptions.IgnoreModuleId` flag exists)
- Audit timestamps (`CreatedTime`, `ModifiedTime`) — informational, not semantic
- Audit user fields (`CreatedBy`, `ModifiedBy`) — informational, not semantic

---

## Effort Estimate

| Component                                                | Manual (1 dev) | AI-Assisted             |
| -------------------------------------------------------- | -------------- | ----------------------- |
| 8 trivial compilers                                      | 3-4 days       | **half day**            |
| 6 simple compilers                                       | 1-2 weeks      | **1-2 days**            |
| 5 moderate compilers                                     | 2-3 weeks      | **2-3 days**            |
| Infrastructure (`IComparisonContext` impl, orchestrator) | 3-5 days       | **1 day**               |
| Testing (unit + integration)                             | 1-2 weeks      | **3-5 days**            |
| **Total**                                                | **~5-8 weeks** | **~1.5-2 weeks**        |

### AI-Assisted Timeline

**Day 1: Design the `IComparisonContext` implementation + extraction pattern**

- Define `CompilerBasedComparisonContext` that collects RPCModels by type
- Establish the extraction pattern on DimensionCompiler (the one with the most entity types)

**Day 2-3: Extract all 18 compilers**

- Each compiler: identify the deserialization calls, collect RPCModels, skip persistence
- The pattern is identical across all compilers — only the entity types and nesting vary
- Trivial + simple ones in a single pass; moderate ones need attention to nesting

**Day 4-5: Infrastructure + wiring**

- `CompilerBasedComparisonContext : IComparisonContext`
- Orchestrator that runs deserialization-only path for both branches → `CompareConfigWorkflow`

**Week 2: Testing + validation**

- Per-compiler tests: deserialization-only path produces expected RPCModels for sample Git content
- Integration: full pipeline (Git → deserialize → compare) against known tenant data
- Regression: verify existing `CompileAsync()` (persistence path) is unchanged

---

## Risks

Implementation-specific risks. For strategic risks (scope expansion, migration, deserialization independence), see [Analysis — Risks](analysis.md#risks-to-flag-for-the-architect) and [Requirements — Risks](requirements.md#risks).

| Risk                                                                                    | Impact       | Mitigation                                                                                                                                                                                                                                                                             |
| --------------------------------------------------------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Breaking the existing Compiler**                                                      | **Critical** | **Do not modify `CompileAsync()` or any existing compiler class.** Build a parallel deserialization path that calls the same static/utility methods (`Util.GetSpecialRpcFromJObject<T>`, `IConfig2Reader.GetEntitiesFromFolder<T>`) without touching production code.                   |
| Diff path drifts from Compiler (new entity type added to compiler but not to diff path) | Medium       | Automated cross-reference test: entity types in diff path vs. `CompareEntityTree` handlers vs. compiler entity types. Any mismatch fails the build.                                                                                                                                    |
| Compare engine entity granularity vs current CM 2.0 granularity                         | Medium       | Adopt CompareEntityTree's finer granularity as authoritative. Derived Git properties (position, `DefaultPageName`) need explicit handling — either exclude from the tree or mark as non-packageable.                                                                                   |

---

## Files Reference

For the full file index (current API, compare engine, compiler, Git structure, proven patterns), see [Analysis — Files Reference](analysis.md#files-reference). Architecture-specific additions:

### Compare Handlers (entity-specific comparison logic)

- `o9.ConfigCompare/Framework/CompareHandlers/` — entity-specific handlers (one per entity type)

### GitSync Adapters (ComparisonNode → Git file mapping)

- `GitSync/Framework/IGitAdapter.cs` — interface: `RpcType`, `GetRelativePath()`, `BuildGitContent()`
- `GitSync/Framework/GitAdapterBase.cs` — base class with AutoMapper (RPCModel → GEModel → Git JSON)
- `GitSync/Adapters/StandardGitAdapter.cs` — standard pattern for folder/file-based entities
- `GitSync/Adapters/` — ~60 entity-specific implementations (DimensionGitAdapter, PlanGitAdapter, etc.)
- `GitSync/TenantConfigGitAdapter.cs` — orchestrator that walks `ComparisonNode` tree and dispatches to adapters

### Package Creation (downstream consumer, to be replaced)

- `Git/api/GitConfigManagementController.cs` — `GetPublishedConfigEntities()` at route `POST /api/v1/config2/publichedConfigEntities/configBlocks/{blockId}/publishedModels/{tenantId}`

---

*See also:*

- *[Requirements](requirements.md) — Problem statement, dependency chain, success criteria*
- *[Analysis](analysis.md) — Codebase investigation and approach comparison*
