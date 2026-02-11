# CM 3.0 — Design Principles

**Origin**: [824295 — CM Import: Reliable Diff Elimination](https://o9git.visualstudio.com/CoreDev/_workitems/edit/824295)

---

## One Authoritative Path Per Data Flow

Every data transformation direction in Config 2.0 has exactly one code path. No duplication, no alternatives.

```
Git content  ──deserialize──►  TenantDB          Compiler
TenantDB     ──serialize────►  Git content        GitSync (via IGitAdapter)
Git content  ──deserialize──►  ComparisonContext   New: CompilerBasedComparisonContext
ComparisonNode ─────────────►  TenantDB           Package import (via CRUD APIs)
User input   ─────────────►  TenantDB           CRUD APIs
```

Each row is a single, authoritative implementation. If two code paths do the same transformation, one of them is wrong — or will be, eventually.

### Why This Matters

CM 2.0's root problems trace back to violating this principle. The current CM package pipeline has its own deserialization path (`GetPublishedConfigEntities`) that re-reads Git content and deserializes independently of the Compiler. Two deserialization paths for the same data means two chances to diverge — and they did. Different serialization order, different default handling, different entity granularity.

The semantic config pipeline enforces this principle:

- **Deserialization** always goes through the Compiler's `GetSpecialRpcFromJObject<T>()` calls — whether loading a tenant or building a comparison context
- **Serialization** always goes through GitSync's `IGitAdapter` implementations — whether syncing a tenant or applying a package import
- **Entity comparison** always goes through `CompareConfigWorkflow` — whether comparing branches, tenants, or backups
- **Entity writes** always go through the existing CRUD controllers — whether from user input, package import, or any future consumer

No new paths. No shortcuts. New features compose existing paths — they don't duplicate them.

---

*See also:*
- *[Requirements](requirements.md) — Problem statement and dependency chain*
- *[Analysis](analysis.md) — Codebase investigation*
- *[Architecture Notes](architecture.md) — Implementation detail*
