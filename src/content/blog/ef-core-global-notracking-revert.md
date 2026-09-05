---
title: "EF Core NoTracking Broke 50 Repository Methods, and I Reverted It"
description: "A one-line 'performance optimization' — global AsNoTracking in EF Core — silently broke every fetch-then-mutate pattern in a 50-method repository layer. Here's the failure mode and why I rolled it back."
pubDate: 2026-09-07
tags: [dotnet, ef-core, postgresql]
draft: false
---

Every EF Core performance guide tells you the same thing: tracking has a cost, and for read-only queries you should call `AsNoTracking()`. Some guides even suggest making it the *default* at the `DbContext` level:

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;
}
```

One line. Free performance. What could go wrong?

## What went wrong

Our repository layer follows a common pattern: fetch an entity, mutate it, save:

```csharp
public async Task PromoteUserAsync(Guid userId, string newRole)
{
    var user = await _context.Users
        .FirstOrDefaultAsync(u => u.Id == userId);

    user.Role = newRole;       // ← mutation
    await _context.SaveChangesAsync();
}
```

With global `NoTracking`, that `FirstOrDefaultAsync` returns a detached entity. The mutation happens on an object the change tracker has never seen. `SaveChangesAsync()` finds nothing to do — no UPDATE statement is ever issued.

**The kicker: no exception.** The method runs green, the API returns 200, and the database silently keeps the old value. Multiply by ~50 repository methods using fetch-then-mutate, and you have an entire application that *looks* like it works while persisting almost nothing.

## Why it slipped past tests

Some tests used `SaveChanges` directly after adding entities (which is unaffected), others asserted on in-memory state without re-querying. The pattern that broke — fetch *from the database*, then mutate, then assert *from the database* — was exactly the pattern the integration tests didn't isolate.

A global behavior change at the `DbContext` level invalidates an assumption baked into dozens of call sites. That's not an optimization; it's a change to your application's semantics.

## The revert

We reverted it. Full revert, single commit, no attempt to salvage it:

```bash
git revert <commit>
```

## What I'd do instead

If tracking overhead is genuinely a measured problem:

1. **Profile first.** Get numbers from a real workload, not a microbenchmark.
2. **Apply `AsNoTracking()` at explicit call sites** — read-only queries where the detachment is obvious and local.
3. **Keep the default tracking.** The ~50 mutating methods stay correct by construction; opt-out beats opt-in for correctness.
4. **Consider `NoTrackingWithIdentityResolution`** if you need identity resolution without full tracking.

The general lesson: defaults that change *semantics* rather than *speed* are never free. `NoTracking` doesn't make reads faster in a way you'd notice while making writes silently slower-but-invisible — it removes behavior that half your codebase quietly depends on.

Performance optimizations should be local, measured, and reversible. A one-liner that breaks 50 methods isn't an optimization. It's an outage wearing a performance hat.
