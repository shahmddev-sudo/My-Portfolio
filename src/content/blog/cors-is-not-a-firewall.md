---
title: "CORS Is Not a Firewall: The API Lockdown That Didn't Lock Anything"
description: "We 'secured' our API with a CORS allowlist and shipped. A curl command from any laptop proved we had protected exactly nothing. What CORS actually does, and the auth changes that mattered."
pubDate: 2022-03-02
tags: [security, api, web]
draft: false
---

This one still makes me wince, which is why it's worth writing down.

The API backed a React SPA. We wanted "only our frontend can call the API," so we configured CORS: `Access-Control-Allow-Origin: https://app.example.com`, allowlist of methods, done. It felt airtight. The browser enforced it — we tested a rogue origin and got blocked. Ship it.

## The demo that ended the illusion

Someone on the team — correctly skeptical — opened a terminal on a completely unrelated machine:

```bash
curl -i https://api.example.com/api/v1/users/me \
  -H "Authorization: Bearer <any-valid-token>"
```

Full 200 response. Every endpoint. No origin header, no preflight, no problem.

## What CORS actually is

CORS is a **browser cooperation mechanism**. It stops a malicious *web page* in a victim's browser from reading responses from your API using the victim's credentials. That's it. It says nothing at all about:

- curl, Postman, scripts, mobile apps — none of them implement CORS, because they aren't browsers
- requests that don't need a preflight (simple GETs sail through the "protection")
- your server's actual authorization decisions

CORS is a fence around the browser's garden. It is not a wall around your API.

## What we should have done (and then did)

1. **Authentication on every route** — no anonymous catch-alls "temporarily" left open
2. **Authorization per resource** — checking that the token's owner may touch *that* object, not just that a token exists
3. **Rate limiting** — because if an attacker can hammer the endpoint, you have a different problem than origin control
4. **CORS kept tight anyway** — defense in depth for the browser-borne attack surface it actually covers

The allowlist stayed. But nobody on the team ever again described it as "locking down" anything.

## The general lesson

Every security control has a precise threat model. When you can't state in one sentence *which attacker, doing what, is stopped by this*, you don't have a control — you have a comfort blanket. CORS's sentence is: "stops other websites from reading your API responses in a user's browser." Say it out loud before you rely on it for anything else.
