---
title: "Feature Flags Are a Code-Smell Generator: Five Bugs We Actually Shipped"
description: "We maintain a required feature-flag inventory doc, and it exists because five distinct bug classes kept coming back. Each one is a case where the flag system itself — not the feature — was the defect."
pubDate: 2026-09-28
tags: [dotnet, architecture, testing, feature-flags]
draft: false
---

Our repo has a rule that every agent must follow: if you add, remove, rename, or change the default of a feature flag, you must edit the flag inventory document *in the same commit*. A post-merge contract check fails the build otherwise.

We didn't invent that rule because we like documentation. We invented it because the same five bug classes kept coming back, and each one is really a story about what a feature flag system does to the code around it.

This post is those five bugs, in the order I'd rank them by how much damage they did.

## Footgun 1: the test factory inherits your real config

This is the one that cost us three tests and no obvious symptom.

`WebApplicationFactory<Program>` — used bare, without any configuration override — reads your **actual** `appsettings.json`. So when you flip a flag's default to `true` for a dev sandbox, your integration tests silently start running a different code path. Nothing fails. The tests just stop testing what they were written to test.

Three integration tests regressed this way. The fix was three commits, and the real fix was one line per test:

```csharp
// The test factory inherits the real appsettings.json
var factory = new WebApplicationFactory<Program>();
// ...flag flipped in appsettings.json → this test now exercises a different path
```

The defensive version forces you to state the world you're testing in:

```csharp
var factory = new WebApplicationFactory<Program>()
    .WithWebHostBuilder(b => b.UseSetting("FeatureFlags:CaseTimelineEnabled", "false"));
```

**The audit you run before flipping any flag:**

```bash
grep -lE "_FlagOff|_WhenFlagOff" tests/VettifyNG.Integration.Tests/Controllers/
```

For every file that matches, confirm it either sets the flag explicitly or injects a stub. If a flag-off test doesn't state the flag state, it isn't a flag-off test — it's a test that passes conditionally.

This generalizes past feature flags. **Any config-driven behavior change needs a test that pins the config value**, or your test suite is one `appsettings.json` edit away from lying to you.

## Footgun 2: two ways to stub the same thing, testing different systems

There are exactly two idiomatic ways to control a flag in a test, and they're not equivalent:

```csharp
// (a) Through configuration — exercises the production binding path
builder.UseSetting("FeatureFlags:ConsultationBookingEnabled", "true");

// (b) Stub the service — bypasses configuration binding entirely
services.AddSingleton(Substitute.For<IFeatureFlags>());
```

Both work. They test different things. (a) exercises `IConfiguration` → options binding → `IFeatureFlags`, which is the code that actually runs in production. (b) skips all of that and only exercises your controller logic.

Our own bug was a flag-gate test written with (b). It passed, and the flag was in fact wired up wrong in production config. The test was green and the feature was broken.

**Rule we settled on:** prefer `UseSetting` for flag-gate tests, because that's the production path. Reach for a stub only when you're specifically testing the dictionary-style lookup (`IsEnabled("SomeKey")`) with a non-standard key.

The general lesson is the one I keep relearning: **a mock at the wrong seam tests the wrong system.** The seam you pick is a design decision about what you believe is worth verifying.

## Footgun 3: a dev-sandbox flag ships to production

Here's the config layering, which is standard ASP.NET Core and is also a trap:

```
appsettings.json                  ← base: what dev and any un-overridden env sees
appsettings.Production.json       ← layered on top in Production
```

A flag set to `true` in the base file **is `true` in production** unless you explicitly set it to `false` in the production file. This is the rule that feels backwards the first time you meet it: *setting a dev flag on requires you to explicitly turn it off for prod.*

Our own audit flagged this as the canonical bug class for a specific feature, and the fix pattern is now mandatory:

```jsonc
// appsettings.Production.json
{
  "FeatureFlags": {
    "SomeDevSandboxThingEnabled": false   // explicit, even though base says true
  }
}
```

The mental model that makes this stick: **think of the production file as an allowlist of what's permitted in prod, not a list of what's different from dev.** Anything absent inherits whatever the base said, and the base is written for the person developing on a laptop.

## Footgun 4: the silent-success escape hatch

This one is a testing anti-pattern I'd like to never see in any codebase again:

```csharp
[Test]
public async Task Get_FlagOff_ReturnsEmptyList()
{
    var resp = await client.GetAsync($"/api/v1/cases/{id}/timeline");
    if (resp.StatusCode != HttpStatusCode.OK) return;   // "flag off, we're fine"
    Assert.Empty(await resp.Content.ReadFromJsonAsync<List<TimelineItem>>());
}
```

Walk through what that `if` does. The test author is hedging against a flag being off, so the test passes whether the endpoint returned an empty list *correctly* or returned a 404 *because the flag happened to be off.* The assertion is dead code half the time.

Then flip the flag to `true` in `appsettings.json` and the test starts running the other branch. If the real behavior is wrong, this test won't catch it — it was never a real assertion.

Our fix was to delete the escape hatch and assert the truth:

```csharp
Assert.Equal(HttpStatusCode.OK, resp.StatusCode);
Assert.Empty(await resp.Content.ReadFromJsonAsync<List<TimelineItem>>());
```

The general rule: **a test that can pass without asserting anything is worse than no test**, because it counts toward coverage while providing none of the safety. If a test's behavior genuinely depends on a flag, set the flag explicitly so there's exactly one branch.

I think of this as the test-suite equivalent of a `catch` block that swallows an exception and returns a plausible default: it converts a loud failure into a quiet one, and quiet is expensive.

## Footgun 5: `IsEnabled("Name")` and the typed property are different fields

This is the one that is purely a design lesson. Our `IFeatureFlags` exposes two ways to read a flag, and they are not aliases of each other:

```csharp
// Typed property — reads a strongly-typed POCO field
if (!flags.CaseTimelineEnabled) return NotFound();

// Dictionary lookup — reads FeatureFlagOptions.Additional["..."]
if (!flags.IsEnabled("CaseTimelineEnabled")) return NotFound();
```

The typed property reads the bound POCO field. `IsEnabled` reads a string-keyed dictionary of "additional" flags. They are **different storage locations**. A flag that exists as a typed property will not satisfy an `IsEnabled("SameName")` lookup, and vice versa.

We shipped exactly that inconsistency: one flag was declared in the typed POCO and read via `IsEnabled()` in a controller. A flag flip appeared to do nothing.

The design lesson is bigger than our bug. **An API that offers two ways to read the same concept is an API that will be used inconsistently**, and the inconsistency will be discovered in production rather than in review, because both forms compile and both look correct.

If you have a typed property *and* a dictionary escape hatch, you need:

1. A documented rule for which to use when (ours: typed property by default; the dictionary only for flags that haven't earned a typed member yet).
2. A follow-up to promote dictionary flags to typed members when they get real usage.

Our current documented rule is exactly that, and the second half — "promote it when it earns the line" — is the part that keeps the escape hatch from becoming the default by inertia.

## Why the inventory doc is mandatory

The rule that started all this: **any change to a flag's name, default, or production override must edit the inventory document in the same commit**, and a post-merge check fails otherwise.

The doc is a table with, for each flag: the default, the production override, the development override, the exact controller gate sites with line numbers, and the user-visible effect in plain language. When a flag is added or removed, the doc's counts and gate sites are audited.

This feels bureaucratic until you consider what the doc actually does: it makes the *blast radius* of a flag change visible. A flag that gates 12 sites across 9 endpoints is a very different thing to flip than one that gates a single check. And the answer to "is this flag still load-bearing?" is right there in the table — a flag with no gate sites listed is a flag that isn't doing anything, which is exactly how two of ours were found to be dead.

The "user-visible effect" column earns its place too. When someone asks "what does `OpinionSealEnabled` actually do," the answer isn't a config key, it's "lets a lawyer seal a final opinion with a cryptographic hash," and the column says exactly that, including the fact that a gate site is still missing.

## The honest summary

Feature flags buy you the ability to ship dark, to disable a broken feature without a deploy, and to run different code in different environments. All real. The cost is that a flag is a *distributed* piece of state that has to agree between config, code, tests, and UI, and every one of those five bugs was a disagreement somewhere in that chain.

The three rules I actually enforce now, distilled:

1. **Pin flag state explicitly in any test that depends on it.** No test reads a flag it didn't set.
2. **Treat the production config as an allowlist.** If a flag is on anywhere, be deliberate about it in prod.
3. **Never leave a `return` in place of an assertion.** A test that can pass silently is a liability.

None of these are about feature flags specifically. That's the part I find satisfying — the flag system is just a particularly good at surfacing general truths about configuration-driven systems: they fail silently, they fail in layers, and they punish "it's just a config value" thinking.
