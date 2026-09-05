---
title: "The Two-Second Crash: Debugging Hangfire.PostgreSql on a Fresh VPS"
description: "Our API container kept dying ~2 seconds after boot on a freshly rebuilt VPS. Caddy 502s everywhere. The culprit was a transaction-enlistment mismatch between Hangfire.PostgreSql and Npgsql."
pubDate: 2026-09-06
tags: [dotnet, hangfire, postgresql, docker]
draft: true
---

We rebuilt a VPS from scratch — new OS image, same docker-compose file, same images. Everything should have been identical. Instead, the API container started dying roughly two seconds after boot, every single time. Caddy returned 502s on every `/api/*` route, and the logs showed the same cryptic crash.

The stack: an ASP.NET Core API on .NET 10, Hangfire for background jobs, `Hangfire.PostgreSql` 1.20+, and Npgsql talking to a `postgres:16-alpine` container.

## The symptom

```text
Unhandled exception. System.ArgumentException:
TransactionScope enlistment must be enabled.
```

The container exited in about two seconds. `docker compose up -d` looked healthy — the service *started* — but by the time traffic arrived, the process was already gone. The health checks never had a chance.

## The diagnosis

`Hangfire.PostgreSql` 1.20+ added a hard precondition: if you disable `EnableTransactionScopeEnlistment` in the Hangfire storage options, then Npgsql must not attempt transaction enlistment. And by default, Npgsql **does**: `Enlist=true` is baked into the connection string unless you say otherwise.

We had `EnableTransactionScopeEnlistment=false` in our Hangfire storage setup — deliberately, since we don't use `TransactionScope` anywhere. But the Npgsql connection string was the plain default:

```text
Host=postgres;Port=5432;Database=appdb;Username=app;Password=...
```

No `Enlist` keyword. So Npgsql enlisted, Hangfire's precondition check fired, and the process crashed at startup — before the web host was even fully up.

## The fix

Append `Enlist=false` to the connection string when it isn't already set:

```csharp
// Program.cs — keep Hangfire and Npgsql on the same page about enlistment
var hangfireConnection = config.GetConnectionString("Hangfire");
if (!hangfireConnection.Contains("Enlist=", StringComparison.OrdinalIgnoreCase))
{
    hangfireConnection += ";Enlist=false";
}

services.AddHangfire(x => x
    .UsePostgreSqlStorage(hangfireConnection, new PostgreSqlStorageOptions
    {
        EnableTransactionScopeEnlistment = false
    }));
```

The nuance: this is a *policy pair*. Whatever Hangfire believes about enlistment, Npgsql's connection string must agree. It's not enough to configure Hangfire alone.

## The second footgun

Here's the part that cost us an extra hour: after fixing the code, the crash *didn't go away* on the VPS. Why? Because:

```bash
docker compose up -d
```

**does not rebuild source-built services.** The compose file builds the API image from a Dockerfile, and `up -d` happily reuses the cached image — our fixed `Program.cs` never made it into the container. The fix was:

```bash
docker compose up -d --build
```

When your bug fix mysteriously "doesn't work" on a server but works locally, check whether the artifact you fixed is actually the artifact that's running. `docker compose up -d` without `--build` is a classic way to fool yourself.

## Takeaways

- `Hangfire.PostgreSql` 1.20+ with `EnableTransactionScopeEnlistment=false` **requires** `Enlist=false` on the Npgsql connection string. They're a matched pair.
- A container that crashes ~2s after boot with Caddy 502s in front is almost always a startup exception, not a network problem. Read `docker logs` before touching the reverse proxy.
- `docker compose up -d` doesn't rebuild. Pass `--build` for source-built services, or better: build in CI and pull pinned tags on the server.
