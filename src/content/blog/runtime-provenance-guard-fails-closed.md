---
title: "A Feature Flag Is Not a Security Precondition — So We Made the Build Check Its Own Version"
description: "A flag was the only thing gating a data-disclosure fix. We replaced documentation with a runtime version gate that fails closed, then spent five review rounds removing its own bypasses."
pubDate: 2026-10-11
tags: [dotnet, security, architecture, testing]
draft: false
---

We had a feature flag standing between a data leak and production, and it was not doing the job.

The endpoint exposed an internal audit log to case participants. The fix — an allow-list of disclosable action names plus redaction of the details column — landed *after* the flag that turned the endpoint on. Nothing checked that relationship, because the dependency lived in a document:

> **The flag is only safe on a build that includes the hardening.**

That is a true sentence. It is also unenforceable. An operator deploying an older image and setting the flag gets a working endpoint with the leak, and no stack trace, no failed test, no health-check failure tells them.

## The gap

| Layer | What it does | Why it failed here |
|---|---|---|
| Documentation warning | Tells operators the order matters | Not code; nothing enforces reading it |
| The flag itself | Decides whether the endpoint serves | Older builds have the flag too |
| The release pipeline | Not wired to flag state | Out of scope of the image |

**A doc warning is not enforcement.** The fix had to become a runtime assertion, which meant the running process had to answer a question about *itself*: does this build contain the fix?

## What the guard reads

One environment variable, stamped at image build time, compared against a hardcoded floor:

```csharp
private const int HardeningMajor = 1;
private const int HardeningMinor = 7;
private const int HardeningPatch = 10;

private static bool TimelineHardeningPresent =>
    DetectTimelineHardening();

private static bool DetectTimelineHardening()
{
    var appVersion = Environment.GetEnvironmentVariable("APP_VERSION");
    return !string.IsNullOrWhiteSpace(appVersion)
        && TryParseAtLeast(appVersion.Trim(), HardeningMajor, HardeningMinor, HardeningPatch);
}
```

The floor lives in named constants rather than a literal tuple so that raising it cannot leave the guard and its tests disagreeing — a `(1, 7, 10)` written in both places is a future bug that compiles cleanly.

The whole decision evaluates per request instead of caching in a `static readonly`. `APP_VERSION` does not change during a container's life, so this costs one environment read — and in exchange the guard stays reachable from a test that drives a real request, rather than only through reflection.

## The bypasses

The first version was correct in intent and wrong in several separate ways, each one found by a different review round. Every one of them is a way to make the guard say yes to a build it should have rejected.

### 1. A commit sha is not a provenance check

The original accepted a `GIT_COMMIT` fallback when `APP_VERSION` was unusable. The argument was that a sha identifies the build. It does not: **commits cannot be ordered at runtime.** There is no way to ask "is this commit an ancestor of the one containing the fix?" from inside the running process, so any implementation is comparing strings, and any 40-character string passes.

> Comparing shas is not a weaker version of a provenance check — it is not a check.

The fallback was deleted rather than tightened. Final semantics:

| `APP_VERSION` state | Result |
|---|---|
| Legible and `>=` the floor | Allow |
| Legible but older | Deny |
| Missing | Deny |
| Unparseable | Deny |

A deployment that cannot prove its own provenance does not serve a feature whose safety depends on it.

### 2. A numeric prefix is not a version

The parser accepted anything that *started* with digits, so `1.7.10garbage` and `1.7.10rc1` both read as `1.7.10` and passed. The value must **be** a version, not merely begin with one. Now build metadata is stripped, and whatever remains must parse as exactly three numeric components — so trailing junk fails naturally.

```csharp
var parts = text.Split('.');
if (parts.Length != 3
    || !int.TryParse(parts[0], System.Globalization.NumberStyles.None,
                     System.Globalization.CultureInfo.InvariantCulture, out var gotMajor)
    || !int.TryParse(parts[1], System.Globalization.NumberStyles.None,
                     System.Globalization.CultureInfo.InvariantCulture, out var gotMinor)
    || !int.TryParse(parts[2], System.Globalization.NumberStyles.None,
                     System.Globalization.CultureInfo.InvariantCulture, out var gotPatch))
{
    return false;
}
```

### 3. Parse styles are a security surface

`NumberStyles.None` is doing real work here, and the default styles would have quietly broken it. Per the .NET API docs:

> Indicates that no style elements, such as leading or trailing white space, thousands separators, or a decimal separator, can be present in the parsed string. The string to be parsed must consist of integral decimal digits only.

Default parsing styles tolerate leading or trailing whitespace and a sign, so a stamp like `1.+7.10` could parse. **A version stamp that needs leniency is not a version stamp.**

### 4. Pre-releases are exactly the builds you least want

`1.7.10-rc1` parses numerically as `1.7.10` and passed the gate. But an rc of a version is cut *before* that version ships, so it is not covered by the guarantee the released build carries — and release candidates are the builds most likely to predate a fix.

This is a genuine ordering fact about pre-releases, not an opinion. SemVer 2.0.0 §11 places a pre-release version below its associated normal version, and §10 states build metadata "MUST be ignored when determining version precedence" — so `1.7.10+build.7` is correctly *equal* to `1.7.10` and allowed, while any `-rc` marker fails the guard.

## Where the check sits in the request

Order is load-bearing here, and getting it wrong reintroduces a leak that looks unrelated:

```csharp
// Authz FIRST — a flag-off short-circuit before this leaked case existence.
if (!isParticipant)
{
    return Forbid();
}

if (!flags.TimelineEnabled)
{
    return Ok(Array.Empty<object>());
}

if (!TimelineHardeningPresent)
{
    return NotFound();
}
```

A flag-off short-circuit placed *before* authorization turns the endpoint into an existence oracle: an unauthorized caller gets an empty `200` for a real case and a `404` for a fake one. The guard itself comes last, because it is a precondition on serving, not an access control.

## Testing the thing that actually matters

The parser had a thorough theory matrix, and it proved nothing on its own — it never proved the controller *consults* the detector. The test that mattered drove a real request through the flagged host and asserted a `404` with no leak payload:

```csharp
// APP_VERSION must be removed AFTER the client is built — the factory
// re-stamps it on a client created earlier, which would model a stamped
// build instead of the unstamped case the test is trying to reproduce.
```

That test also forced the factory to stamp `APP_VERSION` only when unset, because unconditional overwriting made the unstamped scenario unrepresentable. **The guard is read per request from the live process environment, which is exactly why it was never cached in a static.**

Test count across the iterations: 119 → 129 → 133 → 137 → 138.

> A comment-only edit to that factory once dropped the `Environment.SetEnvironmentVariable("APP_VERSION", ...)` call itself, which broke six endpoint tests with `404`. Caught by running the suite rather than trusting the diff.

## What this guard does not do

The honest part, because the overclaim is where this class of fix usually fails review:

- It **cannot** protect an image that predates it. That image has no such check and will serve the feature if the flag is on. Closing that path needs enforcement in the release pipeline, outside the image.
- `APP_VERSION` is **self-attestation**. It defends against deploying a stale image by mistake; it does not defend against an operator lying about the version.
- The real protection was always **ordering** — flip the flag only once the hardened build is live.

## The rule

**When a flag gates a feature whose safety depends on some other change, encode that dependency as code that fails closed — and then read your own guard adversarially, because the first version you write will be a bypass.**

If you cannot prove a build contains a fix at runtime, that is worth knowing *before* you ship the flag, not discovering from an audit.

What I would add next, outside the image: have the release pipeline refuse to enable a flag whose hardening is absent from the tag being deployed. The guard is the floor, not the ceiling.