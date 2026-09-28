---
title: "Why Our Refresh Tokens Rotate on Every Use (and How We Detect a Stolen One)"
description: "A rotating refresh-token design with family-based reuse detection: what each field is for, why the hash algorithm differs from the one we use for passwords, and the exact sequence that traps a replayed token."
pubDate: 2026-09-28
tags: [dotnet, security, architecture]
draft: false
---

Most session bugs are not authentication bugs. They're *revocation* bugs: a token that should have been dead is still accepted, and nobody notices until it matters.

This post is the refresh-token design we settled on for a multi-tenant SaaS, and the specific mechanism that makes a stolen token self-destruct. I'll walk through the data model, because that's where most of the design actually lives.

## The two-token split

Access tokens are **short-lived signed JWTs** — HMAC-SHA256, with claims for `sub` (user ID), `jti` (a random GUID), `role`, and `org_id` where the user belongs to a corporate tenant.

The lifetimes are role-tiered, and this is deliberate:

| Role tier | Access TTL |
|---|---|
| SystemAdmin, CorporateAdmin | **5 minutes** |
| Everyone else | **15 minutes** |

An admin session is the highest-value thing to steal, so it has the shortest window. Fifteen minutes is already generous; five is the number I'd pick for anything that can move money or change roles.

Refresh tokens are **longer-lived and stored server-side**, in an entity I'll walk through field by field, because each field earns its place.

```csharp
public class RefreshToken
{
    public string TokenHash { get; set; } = string.Empty;   // never the raw value
    public Guid FamilyId { get; set; }                       // the rotation chain
    public Guid AccessTokenJti { get; set; }                 // ties to a specific access token
    public DateTimeOffset ExpiresAt { get; set; }            // absolute lifetime
    public DateTimeOffset IdleExpiresAt { get; set; }        // idle timeout
    public bool IsConsumed { get; set; }                     // replaced during rotation
    public bool IsRevoked { get; set; }                      // killed by reuse detection
}
```

## Field by field: why each exists

**`TokenHash`, and the algorithm choice.** The raw refresh token is never persisted. We store a hash and look it up by equality:

```csharp
return await _db.Set<RefreshToken>()
    .FirstOrDefaultAsync(r => r.TokenHash == tokenHash, ct);
```

Here's the nuance worth being precise about, because it's easy to get wrong: this is a **SHA-256** hash, not Argon2id. That's intentional and it's not a shortcut.

Passwords use **Argon2id** — a slow, memory-hard KDF with a per-password salt and constant-time comparison (`CryptographicOperations.FixedTimeEquals`). Passwords *need* that because they're low-entropy and guessed offline.

Refresh tokens are **high-entropy random values**. An attacker with the database can't meaningfully brute-force a 256-bit random token, so the KDF cost buys nothing. What matters is that the raw token never lands in the DB — a dump exposes opaque hashes, not usable credentials.

The honest generalization: **match the primitive to the threat.** Slow KDF for what humans type; a plain digest for what a machine generated. Using Argon2id for refresh tokens would be slower and provide no additional protection.

**`FamilyId` — the rotation chain.** Every login creates a new random UUID. Every refresh keeps the same `FamilyId`. This is the field that makes reuse detection possible, and I'll come back to it.

**`AccessTokenJti`** ties a refresh token to the specific access token it issued. That's what lets us revoke a specific access token rather than the whole session.

**Two expiry fields.** `ExpiresAt` is the absolute lifetime of the token; `IdleExpiresAt` is an idle timeout refreshed on every use. You want both, for different reasons: the absolute cap bounds how long a session can *possibly* live even under continuous use, while the idle timeout kills abandoned sessions quickly. We use a 2-hour idle window and an absolute session cap.

**`IsConsumed` versus `IsRevoked`** — these are different states and conflating them is a bug. `IsConsumed` means "rotated out normally, replaced by a newer token." `IsRevoked` means "killed because something looked wrong." A consumed token is expected; a revoked one is an incident.

## The cookie

The refresh token is delivered as `vtf_refresh`: `HttpOnly`, `Secure`, `SameSite=Lax`, scoped to `Path=/api/v1/auth`.

The `HttpOnly` flag is the load-bearing one: JavaScript cannot read the value, so an XSS that steals your access token still can't steal the refresh token. And scoping the path to the auth endpoints means the cookie isn't sent on ordinary API calls, shrinking the surface where it could leak.

Note `SameSite=Lax`, not `Strict`. `Strict` breaks the legitimate "click a link, land back in the app" flow; `Lax` still blocks cross-site POSTs, which is the CSRF vector that matters here.

## Rotation and the reuse trap

This is the part I want to get exactly right.

**On every refresh:** the presented token is marked `IsConsumed = true`, and a new token is issued in the same `FamilyId`. A consumed token can never be used again — the lookup returns it, and the consumed flag rejects it.

**On reuse:** if a *consumed* token is presented again, that's not a normal request. It means either a client is replaying an old token or an attacker has one. In both cases we **revoke the entire family**:

```csharp
var tokens = await _db.Set<RefreshToken>()
    .Where(r => r.FamilyId == familyId)
    .ToListAsync(ct);

foreach (var t in tokens)
    // revoke every token in the chain
```

Here's why this matters and why the family structure is the whole design. Suppose an attacker steals your refresh token on Monday. On Tuesday you refresh normally — your client gets a new token, the stolen one is consumed. On Wednesday the attacker tries the stolen token. It's already consumed, so the reuse check fires and **kills the family** — including the token you're currently using.

Both of you get logged out. The legitimate user is inconvenienced by one login; the attacker is permanently locked out. That asymmetry is exactly what you want, and it only works because the *family* is the unit of revocation rather than the individual token.

Without `FamilyId`, you'd have to detect "this specific token was already used" and then somehow find and revoke its descendants. With the family, it's one query and one loop.

**One consequence worth planning for:** concurrent tab refreshes each get a new token in the same family, and the stale one triggers reuse detection, logging out that browser. That's the designed trade — it means a genuine replay looks the same as a benign race. If you want to tolerate concurrent refreshes you need a grace window on consumption, and then the reuse detection is weaker. Pick one deliberately.

## Access-token revocation

Access tokens are stateless by design, which means logging out can't invalidate an already-issued JWT. We handle this with a **JTI blocklist in Redis**: on logout or when a rotation family is revoked, the access token's `jti` goes into the blocklist.

The blocklist entries use a TTL matching the maximum access-token window, so the keyspace cleans itself up as sessions end. That's the cheap and correct approach — you only need to blocklist for as long as the token could still be valid.

## The other half of the system

Two things sit alongside this and are worth having regardless of your token design:

**Rate limiting on the auth endpoints.** We rate-limit login attempts IP-scoped, which is what actually defends against credential stuffing. Token design doesn't help if someone can try a million passwords.

**Argon2id for passwords, compared in constant time.** The salt is stored alongside the hash, and verification uses `CryptographicOperations.FixedTimeEquals` to avoid leaking timing information about the stored hash. If your password comparison isn't constant-time, fix that before you touch anything else in this post.

## What I'd tell someone building this

The core insight is small: **make the unit of revocation the session, not the token.** Everything else follows from that. `FamilyId` exists to give you a unit to revoke. `IsConsumed` exists to make a replay detectable. The two expiry fields exist because "active" and "abandoned" are different failures.

And the design decision I'd defend hardest: short access-token TTLs aren't about being paranoid. They're about bounding the damage of the thing that's most likely to actually leak — a token in browser memory, or an access token in a log line. If your access token is 15 minutes of exposure at worst, most token-leak incidents are no longer incidents.
