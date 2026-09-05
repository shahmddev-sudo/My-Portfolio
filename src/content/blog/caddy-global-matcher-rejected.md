---
title: "The Caddyfile That Committed Broken: File-Global Matchers Get Rejected"
description: "A security fix that never actually deployed: a Caddy request matcher defined at file-global scope is invalid Caddyfile syntax, Caddy rejects the whole config, and the bootstrap silently skipped it. Here's the snippet pattern that fixes it."
pubDate: 2026-09-09
tags: [caddy, devops, security]
draft: true
---

The bug was public `/metrics` and `/hangfire` endpoints behind our reverse proxy. The fix was simple: a Caddy matcher that responds 404 for internal paths. It was written, reviewed, and committed.

Then Caddy refused to load it — and here's the part worth remembering: **the config was committed broken, and nothing failed loudly.**

## The invalid config

A request matcher defined at file-global scope, between site blocks:

```caddy
# ❌ invalid — matchers may not be defined globally
@internal_paths path /metrics /metrics/* /hangfire /hangfire/*
```

Caddy adapts the Caddyfile and stops with:

```text
Error: adapting config using caddyfile: request matchers may not be defined
globally, they must be in a site block; found @internal_paths
```

With a broken Caddyfile, the deploy script validated it, warned, and *skipped the install*. The old config kept running. So the security fix existed in git, was "shipped" in a commit, and was never live. The endpoints stayed exposed until someone actually probed them.

## The fix: a named snippet

Caddy supports named snippets — reusable blocks imported into each site:

```caddy
# ✅ named snippet — define once, import everywhere
(internal_paths) {
	@blocked path /metrics /metrics/* /hangfire /hangfire/*
	respond @blocked 404
}

app.example.com {
	import internal_paths
	reverse_proxy localhost:8080
}

alt.example.org {
	import internal_paths
	reverse_proxy localhost:8080
}
```

The matcher now lives *inside* a snippet scope, which Caddy accepts, and each site imports it. Every public hostname gets the 404 behavior.

## Lessons

1. **Validate where you'll deploy, not just where you develop.** The snippet above passes `caddy validate --config Caddyfile --adapter caddyfile` — run it in CI, and a broken file never reaches a server.
2. **A skipped config install is a failed deploy, not a warning.** If your bootstrap script "warns but continues" when validation fails, you've built a machine that silently reverts your security fixes. Fail hard.
3. **Verify the fix from the outside.** After the deploy, the only proof that counts:

```bash
$ curl -i https://app.example.com/metrics
HTTP/2 404
```

The matcher scoping rule is a five-minute fix. The meta-lesson is the one to keep: a committed fix that isn't verified live is indistinguishable from no fix at all.
