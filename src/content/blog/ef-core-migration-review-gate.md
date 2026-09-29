---
title: "EF Core Lessons From a Multi-Tenant SaaS: Provider Drift, Migration Holes, and the Review Gate That Prevents Them"
description: "Six months of EF Core in production produced a written checklist we now enforce on every migration PR. This is the checklist, and the incidents that made each line necessary."
pubDate: 2026-09-28
tags: [dotnet, ef-core, postgresql, architecture]
draft: false
---

This post is a checklist we enforce on every pull request that adds a database migration. Before you skip it as boring process, read the incidents behind each line — every rule in the list exists because something shipped and then broke.

## Rule 1: the migration chain must apply on a fresh database

This is the rule that would have saved the most pain.

We had a migration whose `Up()` method created a table, and the chain in `__EFMigrationsHistory` looked complete. It wasn't. A merge had reverted the model snapshot to a different provider's shape and dropped a migration from the chain entirely.

The symptom was infuriating: fresh databases built correctly in a way that didn't reflect production. The real chain had a hole, and the hole meant a table that should have existed didn't.

The fix involved regenerating the snapshot against the correct provider, deleting the broken migration, hand-writing the additive DDL, and verifying on a genuinely fresh Postgres. And there's a specific trap in hand-written migrations I want to state plainly:

```csharp
[DbContext(typeof(AppDbContext))]
[Migration("20260916120000_AddPayoutRequests")]
public partial class AddPayoutRequests : Migration
```

Both attributes, always. A migration class missing `[Migration]` is simply never discovered and never applied — which is precisely how a "complete" chain can be missing a table. (`[DbContext]` matters when you have more than one context; without it EF can't tell which model the migration belongs to.)

**Now mandatory:** before merge, `dotnet ef database update` (or `MigrateAsync` in an integration test) must succeed against an empty Postgres. Snapshot drift is chain-hole evidence.

One caveat worth knowing: this verifies chain *integrity*, not *populated* behavior. Destructive operations additionally get tested against a `pg_dump` restore of prod-shaped data.

## Rule 2: annotate the direction of every migration

This rule exists because an unstated migration direction silently dropped schema.

We now require every migration's `Up()` to end with an explicit comment stating which operations touch a populated table — add table, add nullable column, add index, type change, rename, drop — and whether `Up()` is purely additive. If there are drops, they must be identified as `Down()`-only or called out as executed inside `Up()`.

Worth being precise about that distinction, because it comes up constantly: a forward `MigrateAsync()` runs each pending migration's `Up()`. `Down()` only runs when you explicitly migrate to an earlier target. So a drop that "lives in `Down()`" is a **rollback path**, not something a normal deploy will execute — which is exactly why the annotation has to say which of the two it is.

The point isn't the comment. It's that the author has to answer the question out loud. A migration that says "this is additive only" is a claim a reviewer can check. A migration that says nothing about direction forces the reviewer to read DDL and infer intent.

## Rule 3: `DropColumn` / `DropTable` / `AlterColumn` in `Up()` blocks merge

Any destructive operation in an *up* migration is a data-loss operation, and we don't merge it until the author has answered two questions in writing:

1. Which production data is deleted or converted?
2. Is there a documented backfill plan?

This is the single highest-value line in the checklist. It's cheap to ask and expensive to skip.

## Rule 4: type changes and renames on live tables use expand/contract

If a table is populated and you need to change a column type or rename one, that ships in **two** migrations: add the new column, backfill it, switch reads over, then drop the old one. A single-shot `AlterColumn` or `RenameColumn` on a populated table requires a stated downtime or locking justification.

This is the pattern everyone knows and nobody applies under deadline pressure. The checklist exists so that skipping it is a visible, deliberate choice rather than an accident.

## Rule 5: big tables need `CREATE INDEX CONCURRENTLY`

On tables expected above roughly a million rows, a `CreateIndex` in a migration body takes a lock. The right call is raw SQL, run outside the transaction:

```csharp
migrationBuilder.Sql(
    "CREATE INDEX CONCURRENTLY IX_Cases_OrgId_Status ON \"Cases\" (\"OrgId\", \"Status\");",
    suppressTransaction: true);
```

`suppressTransaction: true` is the part to understand before you reach for this: **the migration now runs outside a transaction**, which means

- it cannot roll back atomically,
- a failure leaves an **invalid** index behind (Postgres marks it `INVALID`; it is unusable for query planning and must be dropped before you retry),
- and the `__EFMigrationsHistory` row is only written if the script completes.

Plan the retry-and-repair step *before* shipping one. I've seen this bite: a migration that partially succeeds and then doesn't record itself in history is a specific kind of mess.

## Rule 6: your LINQ must translate on *both* providers

We run some environments on SQLite and some on Postgres, which is a legitimate choice for test speed. The cost is that a LINQ query that works on one provider can fail on the other.

Our actual incident, from the login path:

```text
System.InvalidOperationException: The LINQ expression 'DbSet<User>()
    .Where(u => u.Email != null && u.Email.ToLowerInvariant() == @ToLowerInvariant)'
    could not be translated.
```

`ToLowerInvariant()` has no SQL translation in EF Core's SQLite provider. `ToLower()` does — it maps to `LOWER()`. The fix, in the authentication repository, is one method name and an honest suppression:

```csharp
#pragma warning disable CA1862, CA1304, CA1311
return await _db.Users.FirstOrDefaultAsync(
    u => u.Email != null && u.Email.ToLower() == trimmed.ToLower(), ct);
#pragma warning restore CA1862, CA1304, CA1311
```

The `#pragma` is not a hack — the analyzers are correctly telling you that `ToLower()` is culture-sensitive. The comment explains *why* it's correct here: the expression has to survive translation to `LOWER()`, which is what the query needs anyway. Suppressing with a documented reason is better than a "correct" method that can't run.

Two follow-up lessons fall out of this one:

**Test against the real provider.** A unit test that runs the query in memory passes. Our integration suite boots the real context against a Testcontainers Postgres and runs `MigrateAsync`, so the chain and the LINQ both get exercised on every blocking run. That's how this class of bug gets caught before it reaches anyone.

**Provider differences are documentation, not accidents.** If you're going to run two providers, keep the list of translation mismatches somewhere reachable — otherwise every one of them is rediscovered at runtime.

## The rule behind all the rules

Look at what all six have in common: **the failure is silent, and the symptom is far from the cause.** A missing attribute, a reverted snapshot, a method name that doesn't translate — each one compiles, passes local tests, and produces a runtime failure in a completely different layer.

The checklist is really a list of places where EF Core will accept something and quietly do the wrong thing. Migrations that don't run. Chains that look complete. Queries that can't translate. Each line forces one of those silent acceptances to become an explicit, reviewable statement.

And the operational lesson that ties it together: **the migration chain is code that runs against data you care about**, and it should get the same review scrutiny as any other destructive change. We didn't get there by being disciplined people. We got there by writing down what broke.

---

*The checklist described here is a distillation of real incidents: a reverted model snapshot that dropped a migration from the chain, and a repository query that failed to translate on one of two providers. Names and paths are generalised.*
