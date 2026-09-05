---
title: "Virus Scanning a 2-vCPU VPS: Why ClamAV Became a Docker Sidecar"
description: "Running ClamAV for user-uploaded document scanning on a tiny VPS seemed impossible — 1GB RAM for the daemon, plus cold-start signature loads. The architecture that made it work: a ClamAV sidecar container speaking the clamd INSTREAM protocol."
pubDate: 2026-09-08
tags: [dotnet, docker, security, architecture]
draft: true
---

A legal case management SaaS accepts document uploads: PDFs, scanned IDs, evidence files. Whatever lands in object storage has to be virus-scanned before anyone opens it. The production host is a 2-vCPU, 3.8GB RAM Debian box already running the API, PostgreSQL, Redis, and a reverse proxy.

ClamAV is the obvious open-source choice. The obvious *deployment* — installing it on the host — is also the obvious disaster:

- The `clamav-daemon` holds the entire signature database in memory: **~1GB** of RAM, permanently.
- Fresh signatures take **several minutes** to download on first boot; the daemon isn't ready until they're loaded.
- On a box where the API and database already compete for ~3.8GB, a permanent 1GB resident daemon is a resource planning failure, not a security feature.

## Requirements before choosing

- The API image must stay slim — no 700MB of ClamAV inside the app image.
- Signatures must stay warm; scanning must not wait on cold starts.
- The solution must survive VPS rebuilds with zero manual steps.

## The architecture: ClamAV as a sidecar

Instead of host installation or baking ClamAV into the API image, run it as a **Docker sidecar container** on the same compose network:

```yaml
services:
  clamav:
    image: clamav/clamav:1.4
    profiles: ["postgres"]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "clamdcheck.sh"]
      interval: 1m
      timeout: 10s
      retries: 3
      start_period: 10m   # ← signature download takes minutes; don't kill it
```

The API container declares a dependency:

```yaml
  api:
    depends_on:
      clamav:
        condition: service_healthy
```

That `start_period: 10m` matters. On a fresh boot, the container spends its first minutes downloading signatures — a tight healthcheck window kills it before it ever becomes useful.

## Talking to clamd: INSTREAM over TCP

The API doesn't shell out to `clamscan` (fork-per-scan, slow, signature duplication). It speaks the **clamd INSTREAM protocol** over TCP:

```csharp
public sealed class ClamAvVirusScanner : IVirusScanner
{
    // Protocol = tcp, Host = clamav, Port = 3310 (compose service DNS)
    public async Task<ScanResult> ScanAsync(Stream content, CancellationToken ct)
    {
        using var client = new TcpClient("clamav", 3310);
        // INSTREAM: send <length-prefixed chunks>, then a zero-length chunk,
        // then read "stream: OK" or "stream: Eicar-Test-Signature FOUND"
        ...
    }
}
```

A tiny `TcpClient` implementation — no ClamAV SDK dependency, no native libs in the API image. The port stays on the internal compose network; it's never published to the host.

## Why not "just move to cloud storage"?

Because it doesn't solve the problem. GCS/S3 don't scan for malware on upload. Changing *where* files live doesn't change *whether* an infected file can reach a user. The scanning decision and the storage decision are orthogonal — and conflating them is how you end up with an infected file sitting in a bucket, waiting to be served.

Cloud object storage remains a candidate for when local volume outgrows the disk — as a storage change, handled separately.

## What this buys

- **Slim API image** — ClamAV lives in its own container, versioned independently.
- **Warm signatures** — the daemon stays up across API redeploys; API restarts don't re-trigger signature loads.
- **No host pollution** — nothing installed on the VPS itself; the whole stack is reproducible from `docker compose up`.
- **Honest resource budget** — the sidecar's RAM cost is visible in `docker stats`, not hidden in a host daemon.

Scanning uploaded documents is non-negotiable for this kind of product. The question isn't *whether* to run ClamAV — it's *where*. On a small box, the sidecar pattern is the answer I'd pick again.
