---
title: "Why Our Refresh Tokens Rotate on Every Use (and How We Detect a Stolen One)"
description: "A rotating refresh-token design with family-based reuse detection: what each field is for, why the hash algorithm differs from the one we use for passwords, the __Host- cookie contract, and the exact sequence that traps a replayed token."
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

Refresh tokens are **longer-lived and stored server-side**. Here is the actual entity, trimmed to the fields that matter for the design (there are more: `UserId`, `ConsumedAt`, `RevokedAt`, `ClientIp`, `UserAgent`):

```csharp
public sealed class RefreshToken : EntityBase
{
    /// SHA-256 hash of the refresh token value. Raw token is never stored.
    public string TokenHash { get; set; } = string.Empty;

    /// All tokens in a family share this value; reuse of a consumed token
    /// triggers revocation of the entire family.
    public Guid FamilyId { get; set; }

    /// The JTI of the access token issued alongside this refresh token.
    /// Used to blocklist the access token on revocation.
    public string AccessTokenJti { get; set; } = string.Empty;

    public DateTimeOffset ExpiresAt { get; set; }          // absolute expiry of this token
    public DateTimeOffset IdleExpiresAt { get; set; }      // idle timeout, updated each refresh
    public DateTimeOffset AbsoluteExpiresAt { get; set; }  // session cap, from first token in family

    public bool IsConsumed { get; set; }                   // rotated out normally
    public bool IsRevoked { get; set; }                    // killed by reuse detection

    public bool IsActive(DateTimeOffset now) =>
        !IsConsumed && !IsRevoked
        && ExpiresAt > now && IdleExpiresAt > now && AbsoluteExpiresAt > now;
}
```

Note `AccessTokenJti` is a **`string`**, not a `Guid` — it holds a JTI claim value, which is a string on the wire. And note there are **three** expiry fields, not two. That third one is the interesting design.

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

**`AccessTokenJti`** ties a refresh token to the specific access token it issued, which lets us blocklist that access token on revocation.

**Three expiry fields, and why three.** This is the part I'd defend hardest:

- `ExpiresAt` — absolute expiry of this individual token.
- `IdleExpiresAt` — idle timeout, updated on every refresh. Kills abandoned sessions quickly.
- `AbsoluteExpiresAt` — **the session cap, measured from the first token in the family.** Every token in a family shares this value.

That last one is what stops the classic rotation weakness: without it, a client that refreshes continuously can hold a session open indefinitely, because each individual token is short-lived but the *chain* never ends. `AbsoluteExpiresAt` bounds the chain, not the token. `IsActive` checks all three, so a session that has been alive too long dies even if every individual token is technically unexpired.

**`IsConsumed` versus `IsRevoked`** — these are different states and conflating them is a bug. `IsConsumed` means "rotated out normally, replaced by a newer token." `IsRevoked` means "killed because something looked wrong." A consumed token is expected; a revoked one is an incident.

## The cookie, and the `__Host-` contract

The cookie layer is centralized in a singleton called `AuthCookies`, and it's the single source of truth for names and options. There are four cookies: `vtf_access`, `vtf_refresh`, `vtf_csrf`, `vtf_session`.

The part worth understanding is the production naming:

```csharp
public string Refresh => _isDev ? RefreshName : $"__Host-{RefreshName}";
public bool Secure => !_isDev;
```

In production the cookie is **`__Host-vtf_refresh`**. That prefix is a browser-enforced contract with three requirements: `Secure`-only, **host-only (no `Domain` attribute)**, and `Path=/`. The browser rejects a `__Host-` cookie that violates any of them.

Why bother: **it defeats cookie-fixing from sibling subdomains.** Without it, a compromised sibling subdomain can set a `Domain=example.com` cookie that shadows yours. With `__Host-`, no other domain or path can plant a cookie with that name. In development the prefix is dropped, because browsers reject `__Host-` cookies over plain HTTP.

The options builder is explicit about one thing:

```csharp
/// Cookie options consistent with the __Host- contract: Path=/ (required),
/// no Domain (host-only).
public CookieOptions Options(bool httpOnly, SameSiteMode sameSite, DateTimeOffset expires)
    => new()
    {
        HttpOnly = httpOnly,
        Secure = Secure,
        SameSite = sameSite,
        Path = "/",          // __Host- REQUIRES Path=/ — not a scoping choice
        Expires = expires,
    };
```

**`Path=/` is not a scoping decision we made.** It's a requirement of the `__Host-` contract. I had this wrong in the first draft of this post — I wrote that we scoped the cookie to the auth endpoints to shrink its exposure. That's a reasonable-sounding idea and it is incompatible with the prefix we use.

The refresh cookie itself is set as:

```csharp
Response.Cookies.Append(
    authCookies.Refresh,
    result.RefreshToken!,
    authCookies.Options(
        httpOnly: true,
        SameSiteMode.Strict,
        DateTimeOffset.UtcNow + jwtSettings.RefreshTokenLifetime));
```

**`SameSiteMode.Strict`**, not `Lax`. That's a meaningful difference: `Strict` means the cookie is not sent on *any* cross-site navigation, including "click a link from an email and land in the app." For a refresh token that's the right trade — arriving from outside should mean authenticating afresh, not silently resuming a session. (The `vtf_session` marker cookie, which is deliberately JS-visible so the SPA can tell whether to attempt silent refresh, uses `Lax`.)

`HttpOnly` is the other load-bearing flag: JavaScript cannot read the value, so an XSS that steals your access token still can't steal the refresh token.

**One honest note on the lifetime.** The entity's XML comment says the absolute expiry is 7 days; the controller comment says `RefreshTokenLifetime` defaults to 10 days. Those disagree, and the truth is the configured value — the entity comment is stale. I'd rather flag that than pretend it's settled, and it's exactly the kind of drift a code comment can't protect you from.

## Rotation and the reuse trap

This is the part I want to get exactly right.

**On every refresh:** the presented token is marked `IsConsumed = true`, and a new token is issued in the same `FamilyId`. A consumed token can never be used again — the lookup returns it, and the consumed flag rejects it.

**On reuse:** if a *consumed* token is presented again, that's not a normal request. It means either a client is replaying an old token or an attacker has one. In both cases we **revoke the entire family**:

```csharp
var tokens = await _db.Set<RefreshToken>()
    .Where(r => r.FamilyId == familyId)
    .ToListAsync(ct);

foreach (var t in tokens)
{
    t.IsRevoked = true;
    t.RevokedAt = DateTimeOffset.UtcNow;
}
await _db.SaveChangesAsync(ct);
```

(The first draft of this post showed that loop with a `// revoke every token in the chain` comment in place of the actual assignment. It's a small thing, but a snippet that omits the operative line is not a snippet.)

Here's why this matters and why the family structure is the whole design. Suppose an attacker steals your refresh token on Monday. On Tuesday you refresh normally — your client gets a new token, the stolen one is consumed. On Wednesday the attacker tries the stolen token. It's already consumed, so the reuse check fires and **kills the family** — including the token you're currently using.

Both of you get logged out. The legitimate user is inconvenienced by one login; the attacker is permanently locked out. That asymmetry is exactly what you want, and it only works because the *family* is the unit of revocation rather than the individual token.

Without `FamilyId`, you'd have to detect "this specific token was already used" and then somehow find and revoke its descendants. With the family, it's one query and one loop.

**One consequence worth planning for — and it depends on how you consume.** Concurrent tab refreshes are the interesting case. There are two correct ways to handle it:

*Atomic consumption* (compare-and-swap on the row, or a conditional `UPDATE ... WHERE IsConsumed = false` and check rows-affected) guarantees only one request can consume a given token. The second concurrent request fails the CAS, sees the token as consumed, and correctly concludes it's a reuse — logging the family out. That's the strict reading, and it's the safe default: **a replay and a race are indistinguishable, so you get the strict behaviour.**

*A grace window* (accepting a consumed token for a few seconds after consumption, returning the same child token) tolerates benign races at the cost of a real replay window during that grace period. That's a deliberate trade, not a free lunch.

Whichever you pick, the thing to get right is that the check-and-consume has to be **atomic**. Two requests that both read an unconsumed token before either marks it consumed will both succeed and *fork* the family — two parallel chains from one parent, neither of which triggers reuse detection. That silently defeats the entire mechanism. If your consume isn't atomic, the design is decorative.

## Access-token revocation

Access tokens are stateless by design, which means logging out can't invalidate an already-issued JWT. We handle this with a **JTI blocklist in Redis**: on logout or when a rotation family is revoked, the access token's `jti` goes into the blocklist.

The blocklist entries use a TTL matching the maximum access-token window, so the keyspace cleans itself up as sessions end. That's the cheap and correct approach — you only need to blocklist for as long as the token could still be valid.

## The other half of the system

Two things sit alongside this and are worth having regardless of your token design:

**Rate limiting on the auth endpoints.** We rate-limit login attempts IP-scoped, which is what actually defends against credential stuffing. Token design doesn't help if someone can try a million passwords.

**Argon2id for passwords, compared in constant time.** The salt is stored alongside the hash, and verification uses `CryptographicOperations.FixedTimeEquals` to avoid leaking timing information about the stored hash. If your password comparison isn't constant-time, fix that before you touch anything else in this post.

## What I'd tell someone building this

The core insight is small: **make the unit of revocation the session, not the token.** Everything else follows from that. `FamilyId` exists to give you a unit to revoke. `IsConsumed` exists to make a replay detectable. The third expiry field exists because a rotation chain can otherwise live forever.

And the design decision I'd defend hardest: short access-token TTLs aren't about being paranoid. They're about bounding the damage of the thing that's most likely to actually leak — a token in browser memory, or an access token in a log line. If your access token is 15 minutes of exposure at worst, most token-leak incidents are no longer incidents.
