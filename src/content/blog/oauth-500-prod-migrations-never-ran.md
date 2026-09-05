---
title: "When OAuth Login 500s in Prod but Not Locally: The Migration That Never Ran"
description: "A production 500 on every Google/Facebook social login, with a 42P01 'relation does not exist' error buried under classified rejection logs. Root cause: EF Core migrations were gated behind IsDevelopment() and never ran in production."
pubDate: 2026-09-10
tags: [dotnet, ef-core, postgresql, oauth]
draft: false
---

The report came in from production: every social login attempt — Google, Facebook — returned a 500. Locally, the same flow worked perfectly. The logs made it worse: the real error, a PostgreSQL `42P01: relation "audit_logs" does not exist`, was buried under generic "classified" rejection messages, so the 500 looked like an auth failure when it was a **missing table**.

## The setup that caused it

A very common EF Core bootstrap pattern:

```csharp
if (app.Environment.IsDevelopment())
{
    app.ApplyMigrations(); // ← migrations only run in Development
}
```

It feels safe. It's even recommended for local dev loops. But it encodes a hidden assumption: *something else* applies migrations in production. If nothing does, your production schema silently rots behind your code.

We had shipped new features — social auth with an audit-log write on every login attempt — and the audit table existed in dev databases only. Production never ran `UPDATE DATABASE`, so the first `INSERT INTO audit_logs` threw `42P01`, the exception bubbled up as a 500, and the auth pipeline classified it as a login rejection.

## Why the logs misled us

The auth stack wrapped the exception in its own rejection handling. From the outside it looked like:

```text
POST /api/v1/auth/social/google → 500 Internal Server Error
```

The 42P01 was several frames down, logged at a level the request pipeline classified as noise. When your error handling *classifies* failures, make sure schema errors are never classified — they're always real.

## The fix

Two parts:

**1. Apply migrations in production, deliberately.** Options in rough order of preference:

```csharp
// At startup, in all environments — with a guard for non-prod databases
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.Migrate();
}
```

or as a release step in CI/CD (my preference for teams: migrations run in the pipeline *before* the new code receives traffic).

**2. Verify with the real flow, not just health checks.** After the fix:

```bash
$ curl -i -X POST https://api.example.com/api/v1/auth/social/google \
    -H "Content-Type: application/json" -d '{"token":"fake"}'
HTTP/1.1 401 TOKEN_VERIFICATION_FAILED   # ← proper rejection, not 500
```

A fake token *should* get a clean 401. Getting a 500 means your auth pipeline is broken somewhere else entirely. After the fix, all 70 tables existed in prod and the endpoint returned proper 401s.

## Lessons

- `if (IsDevelopment()) ApplyMigrations()` means **migrations never run in production**. That's a decision, make it consciously — and pair it with a CI step or an explicit startup call.
- Verify error classification. A schema error wrapped in "auth rejection" clothing costs you hours of looking in the wrong place.
- The dead-giveaway test for any auth endpoint: **send garbage, expect a clean 401.** If you get a 500, the problem isn't the credentials — it's your pipeline.
