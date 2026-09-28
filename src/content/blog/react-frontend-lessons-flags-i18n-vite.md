---
title: "React Frontend Lessons: How We Consume Feature Flags Without a Feature-Flag System"
description: "The React side of a multi-portal SaaS: why we never built a client-side flag system, the i18n parity test that gates every string change, the Vite 8 manualChunks gotcha, and the auth-guard role rule that surprised me."
pubDate: 2026-09-28
tags: [react, typescript, vite, i18n]
draft: false
---

I wrote earlier about the five feature-flag bugs we shipped. This is the other half: how the React frontend actually consumes flags, plus the frontend decisions I'd defend from the same project.

The stack is React 19 with TypeScript 7 and Vite 8, four separate portals (user, lawyer, corporate, admin) in one SPA.

## Feature flags: there is no client-side flag system

I want to lead with this because it's the decision I'm most sure about, and it's the opposite of what most teams do.

We have a server-side feature flag service. The React app **has no corresponding flag store, no `useFeatureFlag` hook, and no flag state in any client cache.** Instead, the endpoint returns the flag state *in the response body*, and the component renders accordingly.

Here's the actual pattern, from a component that shows a "what changed since your last visit" banner:

```tsx
interface ChangesSince {
  lastVisited: string | null;
  changes: ChangeItem[];
  snapshotCursor: string;
  flagEnabled?: boolean;      // ← the flag travels with the data
}

if (dismissed || !data?.flagEnabled || !data.changes?.length) return null;
```

And a widget on the same page:

```tsx
queryFn: () => apiFetch<{ cases: DigestCase[]; flagEnabled?: boolean }>(
  "/api/v1/cases/my-changes-digest?limit=10"),

if (isLoading || !data?.flagEnabled || !data.cases?.length) return null;
```

The backend decides, and its decision arrives as part of the payload. The component's job is only "render nothing if I'm not supposed to."

**Why I'd make the same choice again:**

1. **One source of truth.** There is exactly one place a flag can be wrong. In the system where the client fetches flags separately, you get a second place — and then a third, when you cache them.

2. **No flag drift window.** A separate flag fetch has a race: the page renders before flags arrive, or flags go stale while a page is open. If the flag rides with the data, there is no window.

3. **The 404 case is already handled.** Our API returns 404 when a flag is off. A component that fetched flags *and* data has to handle "flag is off" as a state; a component that only fetches data handles it as an error path it already has.

4. **Trivially testable.** You test the render by feeding a response with `flagEnabled: true` and one with `false`. No mocking of a flag provider.

**The cost, honestly:** you cannot show or hide *statically* rendered UI before data arrives. Nav items, menu entries, and page routes can't be flag-gated this way without a flag-aware fetch for the shell. We solve that by having the shell fetch a small "what am I allowed to see" payload — still server-driven, just a different endpoint.

And there's a rule that makes the whole thing work: **if a feature is behind a flag, its endpoints are 404.** That means the client cannot accidentally load a feature the user shouldn't see, and a flag flip is a *server* change, not a client deploy.

## The i18n parity test is the most valuable test in the frontend

Our app ships English and Bengali. The repo rule is that every user-visible string goes through `t("key")` and lands in *both* catalogs in the same commit — including headings, buttons, placeholders, validation messages, loading and empty states, tooltips, `aria-label`, and meaningful `alt` text.

Rules don't enforce themselves. This test does:

```ts
function flatten(obj: Record<string, unknown>, prefix = ""): string[] {
  return Object.entries(obj).flatMap(([k, v]) =>
    v !== null && typeof v === "object"
      ? flatten(v as Record<string, unknown>, `${prefix}${k}.`)
      : [`${prefix}${k}`],
  );
}
```

then four assertions:

```ts
it("has no keys missing from the Bengali catalogue", () => {
  const missing = [...enKeys].filter((k) => !bnKeys.has(k));
  expect(missing).toEqual([]);
});

it("has no extra Bengali-only keys", () => {
  const extra = [...bnKeys].filter((k) => !enKeys.has(k));
  expect(extra).toEqual([]);
});

it("has no empty Bengali values", () => { /* walks bn, fails on "" */ });
```

Three of those are worth separate thought.

**Missing keys** is the obvious one — you added a key to English and forgot Bengali.

**Extra keys** is the one people skip, and it catches a different bug: a key deleted from English but left behind in Bengali. Left alone, that catalog grows orphaned entries forever and nobody notices until someone wonders why a string is still translated.

**Empty values** is the subtle one. A key can exist in the Bengali catalog with an empty string, pass both existence checks, and render nothing in Bengali mode. That's a *visible* bug — an empty button label — that existence checks alone can't see. This assertion is what turns "the key is there" into "the key is usable."

Plus a fourth, small but pointed: `expect(bnKeys.has("app_name")).toBe(true)`. That exists because hardcoding the product name instead of `t("app_name")` is a real defect in a localized app, and it's a defect that looks harmless in review.

**The general lesson:** when you have a two-catalog rule, the test needs to check *both directions* and *value quality*, not just "the key exists somewhere." Most parity checks I see in the wild do only the first assertion and call it done.

## Vite 8 changed the manualChunks contract

This one cost me an afternoon and is worth saving you the time.

We split vendor code into stable chunks so that a deploy doesn't invalidate a megabyte of React that never changed:

```ts
rollupOptions: {
  output: {
    manualChunks(id: string): string | undefined {
      if (!id.includes("node_modules")) return undefined;
      if (/[\\/]node_modules[\\/](react|react-dom|react-router|react-router-dom)[\\/]/.test(id))
        return "vendor-react";
      if (/[\\/]node_modules[\\/]@tanstack[\\/]react-query[\\/]/.test(id)) return "vendor-data";
      if (/[\\/]node_modules[\\/](i18next|react-i18next)[\\/]/.test(id)) return "vendor-i18n";
      if (/[\\/]node_modules[\\/]@microsoft[\\/]signalr[\\/]/.test(id)) return "vendor-realtime";
      return undefined;
    },
  },
},
```

**The gotcha: Vite 8 uses rolldown, and `manualChunks` only accepts the function form.** The object form — which every older snippet you'll find uses — is rejected at build time with `TypeError: manualChunks is not a function`. Not a warning, a hard failure.

Two more details in there that are deliberate:

**`react-router` and `react-router-dom` are grouped together.** Splitting them produced an extra shared chunk, because `react-router-dom` re-exports `react-router`. Grouping them gets one clean vendor chunk.

**The regex matches the exact package directory**, with `\\` or `/` so it works under both npm and pnpm layouts. A naive `id.includes("react")` would put half your app in the React chunk.

**Why stable chunks matter:** the whole point is that `vendor-react` is immutable across deploys. If your chunking is nondeterministic, every deploy re-downloads libraries that didn't change, and your CDN cache is doing nothing.

## Pre-compressing assets at build time

We emit `.gz` and `.br` alongside hashed assets during the build:

```ts
compression({ algorithm: "gzip", ext: ".gz", threshold: 1024 }),
compression({ algorithm: "brotliCompress", ext: ".br", threshold: 1024 }),
```

That moves compression out of the request path entirely — nginx or a CDN serves the pre-compressed file with no per-request CPU. For a self-hosted deployment this is one of the cheapest wins available.

## The stale app shell is a service-worker problem, not a cache problem

We had the classic PWA bug: deploy a new version, users keep seeing the old app shell.

The fix is in the build, not the app: a plugin stamps the release version and content hash into `sw.js`'s `CACHE_VERSION`, so **every deploy rotates the service worker's caches.** No "please hard-refresh" instructions, no version nag banner. The cache name changes, so the old cache is simply not the current one.

The principle: **if a stale-state bug is caused by caching, invalidate at the cache layer, not in the UI.** Any user-facing workaround is a workaround users will hit.

## The auth guard's role rule

Our route guard is small, and the interesting part is one line:

```tsx
export default function AuthGuard({ children, requiredRole }: AuthGuardProps) {
  const { isAuthenticated, role, loading, logout } = useAuth();
  if (loading) return <Spinner className="h-8 w-8" />;
  if (!isAuthenticated) return <Navigate to="/login" replace />;
  if (requiredRole && role !== requiredRole
      && !(requiredRole === "CorporateUser" && role === "CorporateAdmin")) {
    return <AccessDenied />;
  }
  return <>{children}</>;
}
```

That `!(requiredRole === "CorporateUser" && role === "CorporateAdmin")` is the deliberate part: **a CorporateAdmin is allowed where a CorporateUser is required**, but not the reverse. It's an implicit role hierarchy encoded as one exception.

I include it because it's the shape of every real authorization rule — the default is "exact match," and the *exceptions* are the domain knowledge. If you find yourself with three or four of those `&&` clauses, it's time for a role hierarchy table rather than a growing boolean.

Two smaller things in the same component worth noticing:

The `loading` check comes first and renders a spinner. If you check `isAuthenticated` before `loading` resolves, you get a redirect to `/login` on first paint for every authenticated user — a flash of the login screen that's technically correct and deeply annoying.

And the denied state is a real UI with a "go home" link and a "sign in again" button, not a blank page or a raw 401. Access-denied is a *state*, and it should look like one.

## What I'd take to the next project

Three things, in order of how much I'd fight for them:

**Let the server decide visibility and ship it with the data.** Client-side flag systems are a second source of truth with a race condition attached. The 404-when-off rule makes this robust.

**Test translations in both directions and check for empty values.** The parity test is 30 lines and it has caught real bugs every time someone forgot the other catalog.

**Make chunks stable and invalidate caches at the cache layer.** Both are infrastructure decisions that pay off every single deploy, and both are cheap to get wrong in ways you only notice in the network tab.

None of these are React-specific, which I've come to think is the sign of a durable decision. The React part is just where the consequence shows up.
