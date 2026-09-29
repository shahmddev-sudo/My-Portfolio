---
title: "AWS vs Azure vs GCP for a .NET SaaS: The Limits That Actually Decide It"
description: "Not another pros-and-cons list. The documented numbers that decide architecture for a Postgres + containers + background-jobs stack: serverless timeouts, the 230-second Azure wall, Cloud Run's CPU billing model, and who patches your database."
pubDate: 2026-09-28
tags: [aws, azure, gcp, dotnet, architecture]
draft: false
---

Cloud comparisons are usually written by people who haven't been paged at 3 a.m. by one. They list "flexible scaling" and "enterprise reliability" and skip the number that actually broke your design.

So this post is built from documented limits instead. Every figure below was read from vendor documentation on 2026-09-28, and I've linked the sources. Where I couldn't verify something from a primary source, I left it out rather than estimating.

The stack I'm optimising for: a containerized .NET backend, managed PostgreSQL, object storage, and a background job worker. That's an extremely common SaaS shape.

## Who patches your database (the thing everyone misstates)

"Fully managed" is routinely over-claimed. All three vendors run the same split:

- **The vendor owns** the host OS, storage layer, engine binary and *minor* versions, backups, failover plumbing.
- **You own** schemas, data, roles, extensions, tuning, and every **major** version upgrade.

That second row is what generates the 2 a.m. pages. At all three, a major version upgrade is customer-initiated and customer-scheduled.

The differences are real, though:

**AWS RDS** has opt-in auto minor upgrades, and it deliberately does *not* chase the newest release — it designates one minor per major as the target, only after it's been "tested and approved by Amazon RDS," applied during *your* maintenance window.

**Azure Flexible Server** is the best-documented of the three, and by a real margin. It does **in-place** major upgrades that "retain the server name and other settings," require no data migration, and don't require connection-string changes. It also ships **Upgrade Validation Checks** — a dry run days ahead of the window that flags unsupported extensions, logical replication slots, prepared transactions, and pending config. When something can't be down, that tool is worth hours.

**Cloud SQL** patches soonest and gives you the most scheduling control, including rollout week, a 1-hour window, and a **90-day deny period** — plus simulated maintenance events you can rehearse against.

### The AWS pressure mechanism nobody mentions

RDS Extended Support lets you run a major version for **up to 3 years** past end-of-standard-support, for extra money. If you are not enrolled, **AWS automatically upgrades you to the next supported major version.**

That's not a free pass with a long fuse — it's a hard deadline enforced by someone else. Every blog post about AWS database upgrades skips this, and it's the single sharpest operational forcing function across all three clouds.

## Serverless compute: the numbers that decide your architecture

This is where I'd focus if you're choosing a platform for .NET specifically.

### AWS Lambda

| Limit | Value |
|---|---|
| Function timeout (sync) | **900 s (15 min)** |
| Async / event-source mapping | **5,400 s (90 min)** — applies to Lambda Managed Instances; standard on-demand functions stay at 15 min |
| Payload, sync | **6 MB** request, 6 MB response |
| Payload, async | **1 MB** |
| Streamed response | 200 MB; 2 MB/s after the first 6 MB |
| Deployment package | 50 MB zipped / 250 MB unzipped; container image 10 GB |
| Scaling | 1,000 execution environments per 10 s per function per Region |

On cold starts, AWS's own documentation: "Cold starts typically occur in under 1% of invocations. The duration of a cold start varies from **under 100 ms to over 1 second**." Cold start time is billed. Provisioned concurrency gets you "double-digit milliseconds."

**The .NET read-across:** that range is language-agnostic, and .NET cold starts are commonly slower than the Python numbers people quote in blog posts. Measure it yourself rather than trusting the headline.

### Azure Functions (Flex Consumption)

| Limit | Value |
|---|---|
| Function app timeout | default **30 min**, max **unbounded** |
| **HTTP response ceiling** | **230 seconds** |
| Max instances | **1,000** (vs 200 classic Consumption) |
| Instance memory | 512 MB / 2048 MB / 4096 MB → 0.25 / 1 / 2 CPU cores |
| Host app-initialization timeout | 30 seconds, not configurable |

**The 230-second wall is the whole story.** The Flex plan advertises an unbounded function timeout, and it's true — but the app cannot respond to HTTP after 230 seconds anyway, because that is "the default idle timeout of Azure Load Balancer." Microsoft's own prescription is to use Durable Functions or defer the work and return immediately.

Compare that with Lambda's 15 minutes. Same conclusion, completely different mechanism, and Azure's is the harder one because you set a generous timeout and still hit a wall you can't configure.

Two operational notes I'd want to know before committing: the docs say that on timeout "the language worker process restarts. For C# apps running in-process, the host process itself restarts." And the **v3 Linux Consumption runtime stops running after 2026-09-30** — that's this week — with the Linux Consumption plan retiring 2028-09-30. Anything you run there needs to be on v4 or Flex.

### Google Cloud Run

| Limit | Value |
|---|---|
| **Service** request timeout | default **5 min**, max **60 min** |
| **Cloud Run Job** task timeout | default **10 min**, max **168 hours (7 days)** |
| vCPU per instance | default 1; min 0.08; max 8 |
| Memory floor | >512 MiB needs ≥0.5 vCPU; >1 GiB needs ≥1 vCPU, **concurrency must be 1** |

**The CPU allocation model is the part that surprises people, and it's the single most important fact in this section if you run background jobs.**

With **request-based billing** — the default — "CPU is only allocated during request processing." Your container is CPU-throttled between requests. With instance-based billing, "CPU is allocated for the entire container" lifetime.

If you deploy a polling background worker (a Hangfire-style loop, or any queue consumer that does work *between* HTTP requests) to Cloud Run under default billing, **it will stall**, because there is no request in flight to allocate CPU. You need instance-based billing or a Cloud Run Job. Neither AWS nor Azure has this exact failure mode, which means this is a trap that will cost you an afternoon to diagnose.

### Verdict on background jobs

None of the three serverless platforms is a natural home for long-running scheduled .NET work.

- **Cloud Run Jobs** is the least bad — 7-day ceiling, retries, checkpoints, and documented support for continuous background work and worker pools.
- **Lambda's** async path is workable for medium jobs, at the cost of a 1 MB payload and no ordering guarantee. Note the 15-minute ceiling applies to standard on-demand functions; the 90-minute figure is for Lambda Managed Instances, so don't plan around it unless you're on that mode.
- **Azure** forces you into Durable Functions because of the 230-second wall. That's an architectural detour, not a native fit.

If you already run Hangfire in a Docker container, the honest answer is usually "keep the worker on containers."

## Object storage

### Consistency

S3 has had strong read-after-write consistency for new objects since 2020, and for overwrites and deletes since December 2020. GCS publishes 99.999999999% annual durability uniformly across all storage classes, with availability varying by class and geography.

### Event notifications, and the rule that follows from them

GCS documents its delivery semantics most explicitly of the three:

- "There is no SLA for delivery time, but notifications are typically delivered within seconds."
- "Cloud Storage may take **up to 30 seconds** to begin sending notifications" after you add a notification config.
- "Once started, Cloud Storage guarantees **at-least-once** delivery to Pub/Sub" — and Pub/Sub is also at-least-once, so **you will get duplicates with different message IDs for the same event**.
- "Notifications are not guaranteed to be published in the order Pub/Sub receives them."
- If delivery consistently fails, "Cloud Storage may delete the notification after **7 days**."

GCP's own guidance is to use the object's `generation` and `metageneration` as preconditions on your update.

**The practical consequence is identical on all three: design every consumer to be idempotent.** Nobody gives you exactly-once. If your upload handler charges a card, appends a row, or sends a notification, guard it yourself.

### Lifecycle tiers have real arithmetic

GCS is the cleanest table, and the only one with **no minimum object size** and no offline retrieval for any class:

| Class | Min storage duration | Early deletion |
|---|---|---|
| Standard | None | No |
| Nearline | 30 days | Yes |
| Coldline | 90 days | Yes |
| Archive | 365 days | Yes |

Azure Blob is explicit about how early-deletion penalties are computed, which makes cost modelling unusually transparent: delete a blob after 21 days in Cool and you're "charged 9 (30 minus 21) days"; delete an archived blob after 120 days and you're charged for **180 days**. Rewriting the object inside the window also triggers the penalty. Archive rehydration "can take **up to 15 hours**."

And one constraint that catches people: "Only storage accounts that are configured for LRS, GRS, or RA-GRS support moving blobs to the archive tier. The archive tier isn't supported for ZRS, GZRS, or RA-GZRS accounts." **If you chose zone-redundant storage for durability, you lose Archive.**

S3's **Intelligent-Tiering** is the easiest default — automatic movement across three access tiers with no retrieval fees, plus two optional archive tiers you must activate, moving untouched objects after 90 days. Glacier Instant Retrieval has a 128 KB minimum object size and a 90-day minimum duration; Flexible Retrieval is "Minutes to 12 hours."

### Egress

GCP publishes the clearest schedule of the three, with the only documented free allowance — **1 GiB/month per destination free**, then tiered pricing: $0.12/GiB for North America in the 1–1,024 GiB band, stepping down to $0.08 at 10 TiB+.

Two honesty notes. The free 1 GiB is per destination, per month, per account — trivial for a real SaaS. And Google's pricing page states: **"Responses to requests count as data transfer out and are charged."** Your API response bytes are billable egress. In a read-heavy workload that is not a rounding error.

**AWS's egress trap is architectural, and it takes two forms.** The first is the NAT Gateway, which bills for every byte that traverses it — so a misconfigured route table can bill internal traffic. The second, and usually larger, is **cross-AZ data transfer**: traffic that crosses an Availability Zone is charged, and you don't need a NAT gateway for that to happen. AWS routes intra-VPC traffic through the local route; it reaches a NAT gateway only when a route table actually sends it there.

So the practical rule is the same, but for a different reason than people usually give: **co-locate your database and your compute in the same AZ**, because that's what avoids cross-AZ charges on the database path. If you can't, the alternative is a NAT gateway you control the routing to — not an accidental one.

## Cost philosophy — the actual differences

**AWS publishes discount ceilings, which is unusual and genuinely useful:**

- Compute Savings Plans — up to **66%** off on-demand, applying to EC2 *and* Fargate *and* Lambda usage. Migrate EC2 → Fargate and keep the discount.
- EC2 Instance Savings Plans — up to **72%**, family + Region scoped.
- Database Savings Plans — up to **35%** across Aurora, RDS, DynamoDB, ElastiCache, and serverless usage.
- SageMaker AI Savings Plans — up to **64%**.

Two gotchas straight from AWS: "The terms of the commitment can't be changed after purchase," and Dedicated Instances are charged $2/hour in *every* Region you have one running, and those fees are not discounted.

**Azure** offers savings plans (spend a fixed dollar amount per hour for 1 or 3 years, applied automatically, with **unused commitment expiring hourly** — no rollover) and reservations (1 or 3 years, deeper discounts, works best with stable predictable usage). Notably, **Azure publishes no headline percentage**, so you're discounting somewhat in the dark. And compute savings plans "don't cover software, networking, or storage charges."

**GCP** has committed use discounts at 1 or 3 years, and — this is the useful bit — **spend-based CUDs apply to eligible usage in any project the billing account pays for.** That project-agnostic property matters more than it sounds if you have many services. Cloud Run and Cloud SQL CUDs both exist. Sustained-use discounts apply automatically to continuously-used compute with no purchase required.

**The philosophy difference in one line each:** AWS maximises flexibility and gives you the most surface to make expensive mistakes; Azure assembles the bill per resource, so inventory discipline matters more than discount-hunting; GCP has the simplest mental model and the least room for surprises.

## Where each cloud genuinely wins — and hurts

### AWS

**Wins:** the largest service catalogue, the deepest published discounts, and mature RDS/Aurora. EventBridge plus S3 events is the most flexible event fan-out of the three. And if your team has AWS muscle memory, that's not a trivial advantage — it's probably the deciding factor for most small teams.

**Hurts:** accidental cost is the default outcome of naive AWS architecture. NAT gateways, cross-AZ traffic, idle non-prod instances, log groups with no retention policy — each costs a little, collectively a lot. Complexity compounds faster than elsewhere; a small team can't hold the whole system in one head. And the extended-support auto-upgrade is a forced deadline you don't control.

### Azure

**Wins:** the lowest-friction path for a C# team — .NET is Microsoft's own runtime, tooling is first-party. Flex Consumption is a genuinely good serverless story with VNet integration, always-ready instances, and 1,000 max instances. **Database for PostgreSQL Flexible Server has the best upgrade path of the three**, full stop: in-place major upgrades keeping your server name and connection strings, plus pre-flight validation checks. Zone-redundant HA is engineered so minor upgrades apply standby-first with promotion.

**Hurts:** resource sprawl, starting the day you use the portal. Too many resource groups, VMs, public IPs, disks, each a small recurring cost. The naming and tagging pain is the direct consequence — if you don't impose conventions and mandatory tags on day one, you'll never get a clean cost breakdown later. And the 230-second HTTP ceiling is a hard constraint that forces architectural detours.

### GCP

**Wins:** **AlloyDB is the most interesting database of the three** for a SaaS with analytical or AI ambitions — a real in-database **columnar engine** (its own planner and execution engine, 30% of instance memory by default, usable on primary or read pool with transparent query forwarding) plus a customized **pgvector** for semantic search. Operational analytics and vector search on the same Postgres-compatible engine, no second system. Cloud Run is the most straightforward container story, and Cloud Run Jobs is the cleanest serverless answer to background work. Networking is best-in-class, with the most honestly published egress pricing. Cloud SQL's maintenance controls — rollout week, 1-hour window, 90-day deny period, simulated events — are the best operationally.

**Hurts:** smaller breadth in enterprise-adjacent features. The long tail of identity, governance, compliance and line-of-business integrations is thinner than Azure's. If your roadmap includes enterprise SSO or fine-grained audit, you'll be assembling it yourself. And more practically: **your team's familiarity.** GCP has the smallest installed base among .NET teams, and the operational surface — projects, billing accounts vs projects, service accounts, org policy, VPC-SC — is genuinely less familiar. That learning curve is a real cost, and it's the most common reason a GCP-first decision gets quietly reversed eighteen months later.

## What I'd actually do

**Pick the team.** For a small team, familiarity beats every feature difference in this post. The operational surface you already know is worth more than AlloyDB's columnar engine.

**If you're on containers already, keep background jobs on containers.** All three serverless platforms fight you on long-running work, and Cloud Run actively punishes polling workers unless you change the billing model.

**Colocate your database and your compute.** This is free money in AWS specifically and merely sensible elsewhere.

**Design every event consumer to be idempotent** the day you write it, not the day you get the duplicate.

**Read the extended-support clause before you commit to RDS.** It's the one deadline in this post that arrives whether you plan for it or not.

---

*Sources, all read 2026-09-28: [Lambda quotas and limits](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) · [Lambda runtime environment and cold starts](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html) · [Lambda SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html) · [RDS PostgreSQL minor version upgrades](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.Minor.html) · [RDS major version upgrades](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_UpgradeDBInstance.PostgreSQL.MajorVersion.html) · [RDS Extended Support](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/extended-support.html) · [AWS Savings Plans plan types](https://docs.aws.amazon.com/savingsplans/latest/userguide/plan-types.html) · [S3 storage classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html) · [S3 event events](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ev-events.html) · [Azure Functions Flex Consumption plan](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan) · [Azure Functions scale](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale) · [Azure PG Flexible Server major version upgrade](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-major-version-upgrade) · [Azure PG maintenance window](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-maintenance) · [Azure Blob access tiers](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview) · [Azure savings plans](https://learn.microsoft.com/en-us/azure/cost-management-billing/savings-plan/savings-plan-overview) · [Cloud Run request timeout](https://cloud.google.com/run/docs/configuring/request-timeout) · [Cloud Run task timeout](https://cloud.google.com/run/docs/configuring/task-timeout) · [Cloud Run CPU allocation](https://cloud.google.com/run/docs/configuring/services/cpu) · [Cloud Run billing](https://cloud.google.com/run/docs/configuring/billing-settings) · [GCS storage classes](https://cloud.google.com/storage/docs/storage-classes) · [GCS Pub/Sub notifications](https://cloud.google.com/storage/docs/pubsub-notifications) · [GCP network pricing](https://cloud.google.com/vpc/network-pricing) · [GCP committed use discounts](https://cloud.google.com/docs/cuds)*
