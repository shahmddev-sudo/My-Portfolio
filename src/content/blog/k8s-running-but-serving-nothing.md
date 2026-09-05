---
title: "The Kubernetes Pod That Was 'Running' and Serving Nothing"
description: "A pod showed green in kubectl, passed its liveness probe, and returned connection refused. The gap between 'the container is up' and 'the app is ready' — and how readiness probes are supposed to close it."
pubDate: 2024-02-27
tags: [kubernetes, devops, reliability]
draft: false
---

The deploy looked perfect. Rolling update, all pods `Running`, `READY 1/1`, zero restarts. Traffic rolled over and the on-call phone lit up: 502s across the board.

The new pods were up. The app inside them was not.

## The race

Our app takes 40–60 seconds to start — connection pool warm-up, config validation, cache priming. The container image started in 2 seconds, so Kubernetes marked the pod Ready immediately and the Service started routing to a process that was still initializing. Liveness probes passed too, because a liveness probe only asks "is the process alive," and it was — just not listening yet.

## Running ≠ Ready

Kubernetes distinguishes three things, and conflating them is the classic pod failure mode:

- **Running** — the container process exists
- **Liveness** — the process is healthy enough to keep (fail = restart)
- **Readiness** — the app can actually serve (fail = pull from load balancing)

We had probes only for the first two.

## The fix

A readiness probe against a real endpoint:

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  periodSeconds: 5
  failureThreshold: 24   # allow up to ~2 min of warm-up
  initialDelaySeconds: 3
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
```

Two details that matter:

1. **`/health/ready` must check dependencies** (database reachable, migrations applied), while `/health/live` must not — if liveness checks the database and the DB blips, every pod restarts simultaneously and you've turned a hiccup into an outage.
2. **The failure threshold math** — warm-up time ÷ period, plus margin. A probe that fails during legitimate startup restarts the pod in a loop; a period that's too long delays recovery.

## What it costs

With correct readiness, a bad deploy no longer serves a single failed request: pods that aren't ready receive no traffic, and a rolling update waits for real readiness instead of process existence. The 502 storm became a non-event.

The general rule I carry from this: **probes encode your definition of "working." If your definition is "the process exists," Kubernetes will faithfully give you a cluster of existing, useless processes.**
