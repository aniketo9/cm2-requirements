# CM 3.0 Semantic Config Pipeline — Analysis

**ADO Work Item**: [824295 — CM Import: Reliable Diff Elimination](https://o9git.visualstudio.com/CoreDev/_workitems/edit/824295)

**About this document.** The PM produced this analysis using Claude Code to investigate the codebase directly — tracing call chains, reading compiler source, and verifying assumptions against actual code. The architect could do all of this themselves; the goal is to meet in the middle — arriving at the architecture conversation with the codebase investigation already done, so the discussion starts from shared ground rather than a discovery phase. It complements the [Requirements](requirements.md) and [Architecture Notes](architecture.md).

---

## Current Pipeline: End-to-End

The current CM package pipeline has five stages. Each stage feeds the next:

```
1. Diff        →  2. File Selection  →  3. Entity Building  →  4. Package Creation  →  5. Import
(Git API)         (User picks files)    (Re-read + deser.)     (Block-scoped pkg)      (CRUD to TenantDB)
```

**Stage 1 — Diff**: `EntityGitController.GetGitDiffsOfBranches()` calls the ADO Git API to get a textual diff of file changes between two branches, scoped to a single config block (`moduleId`). Returns `List<GitEntityCommitInfo>` — file-level changes (add/edit/delete) with commit metadata and file paths. No deserialization, no understanding of entity semantics.

**Stage 2 — File Selection**: The UI presents the file-level diff. Users select which changed files to include in the package. This is an arbitrary subset — any combination of files can be selected independently.

**Stage 3 — Entity Building**: `GitConfigManagementController.GetPublishedConfigEntities()` receives the selected `List<GitEntityCommitInfo>`, re-reads file content from Git via `gitClient.GetGitEntityDiffContents()`, deserializes each file independently, and resolves parentage by querying TenantDB (e.g., for a `DimensionTranslation`, it looks up the parent Dimension's DB record ID by name). Produces a flat `List<PublishedConfigEntity>` where `ParentEntityId` is a DB record ID.

**Stage 4 — Package Creation**: The package builder takes the flat entity list, resolves dependencies, and serializes into a block-scoped `PublishedModelPackage` (single `ModuleId`).

**Stage 5 — Import**: The import workflow reads the package, resolves entities against the target TenantDB, and dispatches CRUD operations to apply the changes.

Why this pipeline doesn't work and why it can't be adapted for the semantic diff is covered in the [Requirements](requirements.md#from-diff-fix-to-pipeline-redesign).

---

## Existing Infrastructure: What We Can Reuse

The new pipeline is not a ground-up rewrite — it's a rewiring of existing building blocks. Nearly every component in the current platform is reused as-is:

| Component | Reused? | How |
|-----------|---------|-----|
| **CompareConfigWorkflow** (diff engine) | As-is | Already compares any two `IComparisonContext` instances |
| **ComparisonNode** (result tree) | As-is | Already carries change types, source/target objects, full hierarchy |
| **CompareEntityTree** (handler hierarchy) | As-is | Already defines all entity types and their comparison logic |
| **IComparisonContext** (data source interface) | Extended | Add one new implementation for compiled Git snapshots |
| **Compiler** (deserialization) | Extracted | Reuse the deserialization calls; skip DB persistence |
| **GitReader / MergeModules** (base+delta merging) | As-is | Already merges module content before deserialization |
| **CRUD controllers** (entity APIs) | As-is | Import workflow dispatches to existing controllers |
| **TenantBranchSyncWorkflow** (pattern) | Pattern reused | Proven compare-engine-at-solution-level pattern |

The only genuinely new code is: (1) a new `IComparisonContext` implementation backed by the Compiler's deserialization output, (2) deserialization extraction from the existing compilers, (3) the solution-level package model, and (4) the import workflow that dispatches `ComparisonNode` changes to existing CRUD controllers.

### 1. CompareConfigWorkflow — The Diff Engine Already Exists

`o9.ConfigCompare.Framework` contains a production-proven compare engine. It's used today for:
- Config backup comparison (`ConfigCompareController`)
- Git sync change detection (`TenantBranchSyncWorkflow`)
- Config audit after restore (`TenantRestoreConfigCompareAuditHandler`)
- Supportability checks (`SupportabilityCheckExecutor`)

How it works:
- Walks a deeply nested **handler tree** (`CompareEntityTree`, rooted at `TenantCompareHandler`) covering all entity types
- For each entity type, matches source vs target entities by GUID or name
- Compares matched entities **property-by-property** on the deserialized objects (not JSON text)
- Automatically excludes `Id`, `TenantId`, and `[ExcludeFromCompare]` properties
- Produces a `ComparisonNode` tree with change types (Added/Removed/Modified/None) and both source and target objects

Key: `ComparisonHelper.AreObjectsEqual()` compares via `JsonConvert.SerializeObject(propertyValue)` on each individual property — not on the original file content. This is what guarantees that two objects representing the same logical state compare as equal regardless of how they were serialized.

### 2. IComparisonContext — The Data Source Abstraction

The compare engine doesn't care where the data comes from. It reads entities through `IComparisonContext.ListEntities<T>()`. Two implementations exist:

| Implementation | Data Source | Used By |
|----------------|-------------|---------|
| `BackupBasedComparisonContext` | Backup ZIP files (JSON on disk) | ConfigCompare, TenantBranchSync |
| `GlobalDbBasedComparisonContext` | TenantDB (EF Core → AutoMapper → RPCModels) | Supportability checks |

**We need a third**: one that reads from the Compiler's deserialization output (RPCModels collected in memory). This is the primary new code to write. Adding a third data source also generalizes the framework to any-to-any comparison (branch vs. branch, branch vs. tenant, etc.) — see [Future Opportunities](future-opportunities.md).

### 3. The Compiler — Deserialization Is the Same Code Path

The `Compiler` class (`Git/Compiler/Compiler.cs`) is the single source of truth for deserializing Git file content into the platform's entity model. It orchestrates 18 individual compilers (DimensionCompiler, PlansCompiler, UiConfigCompiler, etc.) that each handle specific entity types.

Each compiler follows the same pattern:
```
1. Deserialize: GetSpecialRpcFromJObject<T>(content.EntityJObject) → RPCModel
2. Persist: dimensionCrud.PostDimensionAsync(rpcDim) → DB write
3. Cache: EntityNameIdCache.AddDimensionId(...) → store DB-generated ID for later compilers
```

Steps 2 and 3 exist solely for DB persistence. The diff only needs step 1.

### 4. GitReader — Base/Delta Module Merging

Git file content is not one-file-per-entity in all cases. The `ModuleType` enum defines base (`Common`, `Regular`) and delta (`CommonDelta`, `RegularDelta`) module types. Delta modules have a `ParentModule` reference pointing to a tag in the base module's repo.

**The Compiler uses MergeModules through GitReader.** The call chain is:

```
Compiler.CompileAsync()
  → new GitReader(...)
  → gitReader.LoadAllModulesAsync() / LoadModulesAsync()
      → ExtractDependencies()        — resolves parent/child modules via DFS, extracts ZIP content
      → MergeModules()               — consolidates all module content into a single merged dictionary
          → MergeDeltaAndBaseModuleContent()  — handles rule-specific merging (NamedSets, ActiveRules, InverseRules, Procedures)
      → stores merged result in gitReader._gitContentReader
  → Compiler passes gitReader to individual compilers (DimensionCompiler, PlansCompiler, etc.)
  → Each compiler reads merged content via gitReader.GetJObjectFromFile(), GetContent(), GetAllContentsInDirectory()
```

`MergeModules()` processes modules in dependency order (parent before child). For each module:

- **Integration blocks (Int3)**: stored separately without merging — accessed via `GetAllInt3ContentsInDirectory()`
- **All other modules**: merged into a single content dictionary. Later modules override earlier ones — if a file exists in both base and delta, the delta wins
- **Rules files**: special recursive merge via `MergeDeltaAndBaseModuleContent()` — delta rules inherit from and extend base rules rather than fully replacing them

By the time any individual compiler sees the content, it's already fully merged. The compiler has no awareness of base vs. delta — it sees a single consolidated set of files.

This is critical for the diff: two branches can represent the same entity with different file structures (base-only vs. base+delta), but after `MergeModules()` + deserialization, the RPCModel is identical. The compare engine never knows the difference.

### 5. TenantBranchSyncWorkflow — The Proven Pattern

`TenantBranchSyncWorkflow` already uses `BackupBasedComparisonContext` + `CompareConfigWorkflow` to detect changes for Git sync. The pattern:

```
snapshot1 (previous backup) → BackupBasedComparisonContext
snapshot2 (current backup)  → BackupBasedComparisonContext
→ CompareConfigWorkflow.Compare()
→ BuildGitChanges() → commit to Git
```

This proves the compare engine works end-to-end at the solution level. The new diff follows the same pattern, but instead of backup ZIPs, the data source is the Compiler's deserialization output.

---

## Deserialization Is Independent of DB Persistence

Each compiler interleaves three operations: deserialization, DB persistence, and ID caching. The critical question is whether deserialization can run without the other two.

**Deserialization takes no DB IDs as input.** Every deserialization call follows the same pattern:

```
rpcDim = GetSpecialRpcFromJObject<Dimension>(content.EntityJObject);   // ← JSON in, RPCModel out
rpcDim.Id = 0;                                                         // ← clear DB ID
rpcDim.TenantId = TenantId;                                           // ← set metadata
```

`GetSpecialRpcFromJObject<T>()` takes a `JToken` and returns a strongly typed RPCModel. No DB IDs, no `EntityNameIdCache`, no prior compiler output is involved.

**`EntityNameIdCache` is exclusively a persistence concern.** The cache is:
- **Written** after CRUD calls: `entityNameIdCache.AddDimensionId(context, TenantId, rpcDim.DimensionName, rpcDim.Id)` — where `rpcDim.Id` is the DB-generated ID returned from `PostDimensionAsync`
- **Read** before CRUD calls on child entities: `rpcAttTrans.DimensionAttributeId = entityNameIdCache.GetAttributeId(...)` — to populate foreign keys needed for the DB write

There are 50+ `entityNameIdCache.Get*()` calls across compilers (AlertCompiler, DimensionCompiler, PlansCompiler, UiConfigCompiler, RulesCompiler, GraphCompiler, etc.). Every one of them sets a foreign key property on the RPCModel **after** deserialization and **before** DB persistence. None are inputs to deserialization itself.

**The compare engine doesn't see these foreign keys.** Specifically:

| Concern | Needed for Persistence? | Needed for Diff? | Why |
|---------|------------------------|------------------|-----|
| `EntityNameIdCache` | Yes | **No** | Entity matching uses GUID, not DB ID |
| `Id` / `TenantId` properties | Yes | **No** | Excluded by `ComparisonHelper.AreObjectsEqual()` |
| Foreign keys (e.g., `DimensionId` on Attribute) | Yes | **No** | Both branches have `0` → compare as equal |
| Sequential write phases | Yes | **No** | Handler tree defines hierarchy, not DB relationships |
| CRUD controller business logic | Yes | **Partially** | See risks |

**One exception: `GetStableNewGuid()`.** This runs between deserialization and persistence to assign deterministic GUIDs to entities. It's not a DB operation — it generates GUIDs from entity metadata (module ID, entity type, entity name). Since the compare engine matches entities by GUID, this must be included in the extraction path.

---

## Approaches Compared

### Approach A: Load-then-Compare (Zero Refactoring)

Use the Compiler as-is: load both branches into temporary tenant DBs, then compare via `GlobalDbBasedComparisonContext`.

- **Pro**: Exact same code path as production. Zero risk of deserialization divergence.
- **Con**: Two full solution loads with DB writes — expensive and slow.

### Approach B: Load-then-Backup-then-Compare

Load to temp DB, take backup, compare via `BackupBasedComparisonContext`. Follows the `TenantBranchSyncWorkflow` pattern.

- **Pro**: Proven pattern. Backup format normalizes data.
- **Con**: Still requires two full solution loads with DB writes.

### Approach C: Extract Deserialization Only (Recommended)

Extract just the deserialization calls from each compiler, collect RPCModels in memory, feed to compare engine. Skip all DB writes, ID caching, and cross-entity resolution.

- **Pro**: No temp DB needed. Fast. Clean separation.
- **Con**: Small risk that CRUD controllers apply semantically meaningful transformations during writes that the diff path would miss (mitigatable — see risks).

**Recommendation: Approach C.** Deserialization is cleanly separable from persistence. DB IDs are invisible to the compare engine.

---

## Risks to Flag for the Architect

| Risk | Impact | Notes |
|------|--------|-------|
| CRUD controllers apply defaults/validation during writes that affect the RPCModel state seen by `GlobalDbBasedComparisonContext` | Medium | The diff path sees pre-write RPCModels; the current compare paths see post-write RPCModels. If CRUD controllers mutate entity state in semantically meaningful ways, those mutations won't be present in the diff path. Need to audit which transformations matter. |
| `GetStableNewGuid()` runs between deserialization and DB write | Medium | This is GUID fixup — it's part of deserialization semantics, not persistence. Must be included in the extraction. |
| Missing entity types in extraction | Low | Cross-reference against `CompareEntityTree` handler list to ensure all types are collected. |

---

## Files Reference

### Current API (to be replaced)
- `Git/api/EntityGitController.cs` — `GetGitDiffsOfBranches()` at `GET /api/config/git/gitdiffs`

### Downstream Consumer (to be replaced)
- `Git/api/GitConfigManagementController.cs` — `GetPublishedConfigEntities()` at `POST /api/v1/config2/publichedConfigEntities/configBlocks/{blockId}/publishedModels/{tenantId}`
- `Git/gitapi/PublishedEntity.cs` — `GetPublishedConfigEntitiesAsync()`, `GetEntityAsync()` — the per-file deserialization + DB parentage lookup logic

### Compare Engine
- `o9.ConfigCompare/Framework/CompareConfigWorkflow.cs` — orchestrator
- `o9.ConfigCompare/Framework/ComparisonNode.cs` — result tree node
- `o9.ConfigCompare/Framework/EntityCompareHandlerBase.cs` — entity matching + comparison
- `o9.ConfigCompare/Framework/ComparisonHelper.cs` — property-level equality
- `o9.ConfigCompare/Framework/IComparisonContext.cs` — data source interface
- `o9.ConfigCompare/Framework/CompareConfigOptions.cs` — strategy and options

### Existing IComparisonContext Implementations
- `crud/ConfigBackupRestoreAndDelete/Framework/BackupBasedComparisonContext.cs` — reads from backup files
- `crud/ConfigBackupRestoreAndDelete/Framework/GlobalDbBasedComparisonContext.cs` — reads from TenantDB

### Compiler (deserialization source)
- `Git/Compiler/Compiler.cs` — `CompileAsync()` orchestrator
- `Git/Compiler/CompilerBase.cs` — shared deserialization utilities
- `Git/Compiler/EntityNameIdCache.cs` — DB ID cache (persistence-only, skip for diff)

### Git File Structure
- `Git/Repository/GitReader.cs` — `MergeModules()`, `MergeDeltaAndBaseModuleContent()`
- `Git/GitApiModels/Module.cs` — `ModuleType` enum, `ParentModule` relationship

### Proven Pattern
- `GitSync/TenantBranchSyncWorkflow.cs` — `BackupBasedComparisonContext` + `CompareConfigWorkflow`

---

*See also:*
- *[Requirements](requirements.md) — Problem statement, dependency chain, success criteria*
- *[Architecture Notes](architecture.md) — Implementation-level detail*
