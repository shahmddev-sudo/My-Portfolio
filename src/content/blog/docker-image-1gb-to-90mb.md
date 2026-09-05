---
title: "Our Docker Images Were 1.2GB. Here's the Exact Trail Down to 90MB."
description: "A real image-slimming exercise, step by step: base image choice, layer order, multi-stage builds, and the two surprises (apt cache and .dockerignore) that mattered more than anything fancy."
pubDate: 2023-06-19
tags: [docker, devops, performance]
draft: false
---

When CI minutes and deploy times start hurting, "why is this image 1.2GB" stops being a curiosity and becomes a task. This is the actual reduction trail for one of our services, with the numbers at each step — because the *order* of wins surprised me.

## The starting point: 1.2GB

A .NET web API, built like most first Dockerfiles: `FROM ubuntu:22.04`, apt-get a runtime, copy the published output, go.

## Step 1: Change the base (1.2GB → 780MB)

Swapping to the vendor runtime image (`mcr.microsoft.com/dotnet/aspnet:8.0`) dropped a third instantly. We were shipping a JDK, three text editors' worth of locale data, and a full apt toolchain we never used.

## Step 2: The .dockerignore nobody wrote (780MB → 310MB)

`COPY . .` had been cheerfully including `node_modules` from a stray frontend folder, the `.git` directory (200MB of history in every image!), old build artifacts, and test fixtures. A 20-line `.dockerignore` was the single biggest win of the whole exercise. **It's always .git and node_modules. Always.**

## Step 3: Multi-stage build (310MB → 140MB)

Build in the SDK image, publish to the runtime image:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

The SDK, NuGet cache, and obj/bin intermediates never touch the final image.

## Step 4: Layer order (140MB → 90MB)

The last ~50MB was churn: `COPY . .` invalidated every later layer, so any source change re-shipped everything. Reordering — copy the project files, restore, *then* copy source — means dependency layers stay cached and rebuilds only ship the delta.

```dockerfile
COPY MyApi.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app
```

## What didn't matter

Alpine base swaps, distroless experiments, fancy squashing — together they bought maybe 15MB and cost a weekend of libc-related debugging. The boring stuff (base image, .dockerignore, multi-stage, layer order) was 98% of the win.

If your images are heavy: run `docker history your-image` first. The biggest layers will embarrass you immediately.
