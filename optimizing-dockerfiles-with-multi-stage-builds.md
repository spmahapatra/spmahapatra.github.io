---
title: "Optimizing Dockerfiles with Multi-Stage Builds"
slug: "optimizing-dockerfiles-with-multi-stage-builds"
date: "2026-09-20"
description: "Shrink container images from 1.85 GB to 28 MB using Docker multi-stage builds, BuildKit cache mounts, and non-root Distroless runtimes."
tags: ["docker", "devops", "containers", "performance"]
status: published
section: 7
topic: containers
prev_slug: dockerising-and-standardising-compose
next_slug: caching-api-calls-efficiently
tested_on:
- Docker Engine 26.1
- BuildKit v0.14
- Go 1.22 / Distroless Debian 12
cover_image: https://spmahapatra.github.io/recordings/optimizing-dockerfiles-with-multi-stage-builds/architecture.svg
platforms: [devto, medium, github]
published_at:
  devto: null
  medium: null
  github: https://spmahapatra.github.io/optimizing-dockerfiles-with-multi-stage-builds
---

# Optimizing Dockerfiles with Multi-Stage Builds

Single-stage Dockerfiles routinely ship 1.5 GB+ of compilers, C headers, package managers, and source trees into production just to run a 25 MB binary. During autoscaling events, bloated images stall pod startup in `ContainerCreating` for minutes, exhaust node disk capacity (`KubeletHasDiskPressure`), and leave shells and package managers exposed inside the runtime container.

This blueprint restructures a legacy 1.85 GB single-stage build into a **28.4 MB non-root Distroless multi-stage Dockerfile**, separating compilation dependencies from the execution image and wiring BuildKit `--mount=type=cache` directories so repeat CI builds finish in under 2 seconds.

> [!INFO]
> **What This Blueprint Delivers:**
> - **98% Image Size Reduction:** Drops runtime footprint from `1.85 GB` (`golang:1.22-bookworm` + toolchain) to `28.4 MB` (`gcr.io/distroless/static-debian12:nonroot`).
> - **Sub-2-Second Rebuild Cache:** Uses BuildKit `--mount=type=cache` for module and compiler caches without baking cache directories into image layers.
> - **Zero-Shell Non-Root Hardening:** Runs as UID/GID `65532:65532` with no shell, `apt`, or compiler binaries in the final OCI artifact.

## Architecture Overview

Container images consist of immutable read-only layers; deleting compilers in a later `RUN` instruction still leaves those megabytes inside earlier layers. Multi-stage builds solve this by running compilation inside a throwaway `builder` stage (with persistent BuildKit cache mounts) and copying only the stripped static binary and CA certificates into a clean runtime base:

![Optimizing Dockerfiles with Multi-Stage Builds Architecture Overview](https://spmahapatra.github.io/recordings/optimizing-dockerfiles-with-multi-stage-builds/architecture.svg)

## Project Structure & Prerequisites

- **Docker Engine 23.0+ with BuildKit Enabled:** BuildKit is the default builder in Docker 23.0+; verify with `docker buildx version`.
- **Static Compilation Targets:** Go (`CGO_ENABLED=0`), Rust, or Python wheels isolated from build-time C compilers.
- **Companion Reference Repository:** [`spmahapatra/rick-morty-api-helm`](https://github.com/spmahapatra/rick-morty-api-helm).

```plaintext
# file: Directory Layout
service-repo/
├── Dockerfile.legacy            # Anti-pattern: single-stage 1.85 GB image with SDK + shell
├── Dockerfile                   # Multi-stage BuildKit definition -> 28.4 MB Distroless
├── .dockerignore                # Excludes .git, local caches, and .env from build context
├── go.mod                       # Dependency manifest copied before source tree
├── go.sum                       # Cryptographic checksums for deterministic module cache
└── main.go                      # Application entry point compiled to /build/bin/server
```

## Auditing Layer Bloat in Single-Stage Images (`docker history`)

At 03:14 UTC during a production traffic spike, two Kubernetes nodes threw `KubeletHasDiskPressure` warnings (`Disk usage on image filesystem is at 89%`) and evicted payment pods while replacement pods timed out in `ContainerCreating` (`Failed to pull image: context deadline exceeded`). Build the legacy single-stage image and inspect its layer breakdown with `docker history`:

```bash
docker build -t app:legacy -f Dockerfile.legacy .
docker history app:legacy --format "table {{.ID}}\t{{.Size}}\t{{.CreatedBy}}"
```

```plaintext
IMAGE          SIZE      CREATED BY
d3f2a1b4c5e6   480MB     /bin/sh -c go build -o /app/server .
8f12c9a04b11   720MB     COPY dir:a83b12... in /src
3e91a0c7b402   642MB     /bin/sh -c apt-get update && apt-get install -y gcc git ...
```

And exclude `.git`, local test artifacts, and `.env` files from the build context first so `COPY . .` never invalidates the layer cache on unrelated file edits or leaks secrets into intermediate layers:

```gitignore
# file: .dockerignore
.git
.gitignore
.env
.env.*
*.md
tests/
tmp/
```

## Isolating the Builder & BuildKit Cache Mounts (`Dockerfile`)

Create a named `FROM golang:1.22-bookworm AS builder` stage that copies `go.mod` and `go.sum` before the source tree, mounts BuildKit persistent cache directories (`/go/pkg/mod` and `/root/.cache/go-build`), and compiles a stripped static binary (`CGO_ENABLED=0`, `-ldflags="-s -w"`):

```dockerfile
# file: Dockerfile
# syntax=docker/dockerfile:1.7

# Stage 1: Throwaway Build & Compilation Workspace
FROM golang:1.22-bookworm AS builder

WORKDIR /build

# Copy dependency manifests first so source edits do not invalidate module cache
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

# Copy application source and compile a stripped static ELF binary
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /build/bin/server .

# Stage 2: Minimal Non-Root Distroless Runtime
FROM gcr.io/distroless/static-debian12:nonroot

WORKDIR /app

# Copy only the compiled static executable from the builder stage
COPY --from=builder /build/bin/server /app/server

EXPOSE 8080
USER 65532:65532

ENTRYPOINT ["/app/server"]
```

> [!TIP]
> **Targeted Stage Testing:** You can build and test the compilation stage in isolation during CI linting without running the final stage by passing `--target builder`: `docker build --target builder -t app:builder-test .`.

## Benchmarking Footprint & Rebuild Latency (`docker build`)

Build the multi-stage image with BuildKit enabled, confirm its non-root runtime user (`65532:65532`), and compare its final disk footprint against the legacy single-stage image:

```bash
DOCKER_BUILDKIT=1 docker build -t app:optimized .
docker inspect app:optimized --format='User={{.Config.User}} Entrypoint={{.Config.Entrypoint}}'  # Expected: User=65532:65532 Entrypoint=[/app/server]
docker image ls | grep -E "REPOSITORY|app\s+(legacy|optimized)"
```

```plaintext
REPOSITORY   TAG         IMAGE ID       CREATED          SIZE
app          legacy      d3f2a1b4c5e6   10 minutes ago   1.85GB
app          optimized   a1b2c3d4e5f6   12 seconds ago   28.4MB
```

> [!WARNING]
> **Caveats & Impact:** Because `distroless/static-debian12:nonroot` removes `/bin/sh` and `apt`, you cannot debug a running container with `docker exec -it <container> sh`. On Kubernetes 1.23+, attach a temporary diagnostic sidecar with `kubectl debug -it <pod> --image=busybox:1.36 --target=<container>`. To revert to the legacy image or clean up dangling builder cache layers locally, run `docker tag app:legacy app:optimized && docker builder prune -f`.

## Selecting the Right Runtime Base Image Strategy

Choose the final `FROM` runtime base according to whether your compiled binary links dynamically against `glibc` or runs as a self-contained static executable:

| Runtime Base Image | Final Footprint | Dynamic `libc` | Shell / `apt` | Best Fit |
|---|---|---|---|---|
| `gcr.io/distroless/static-debian12:nonroot` | ~2–28 MB | No (Static only) | None | Static Go / Rust binaries (`CGO_ENABLED=0`) |
| `gcr.io/distroless/base-debian12:nonroot` | ~20–45 MB | Yes (`glibc`) | None | C/C++, CGO, or Rust binaries needing `glibc` & OpenSSL |
| `debian:bookworm-slim` | ~75–140 MB | Yes (`glibc`) | Yes (`bash`, `apt`) | Python (`slim`) or Node.js services needing POSIX shell tools |
| `scratch` | 0 MB base | No (Static only) | None | Ultra-minimal static binaries (must manually copy CA certs) |

## What This Setup Does Not Do & Troubleshooting

Multi-stage builds eliminate build-time bloat from the final image, but they do not automatically persist BuildKit cache mounts across temporary CI runners unless you export cache metadata (`--cache-to type=gha` or `type=registry`), nor do they populate root TLS certificates when you use `FROM scratch` instead of `distroless`. Use this troubleshooting table to fix common multi-stage build errors:

| Symptom / Error | Root Cause | Fix & Verify |
|---|---|---|
| Runtime container exits immediately with `exec /app/server: no such file or directory` on Alpine or Scratch | Binary was compiled dynamically against `glibc` (`/lib64/ld-linux-x86-64.so.2`), which does not exist in `musl` (Alpine) or `scratch` | Set `CGO_ENABLED=0` in the builder `go build` command or switch the runtime base to `gcr.io/distroless/base-debian12:nonroot` |
| Outgoing HTTPS calls fail with `x509: certificate signed by unknown authority` | Final stage uses `FROM scratch` without copying `/etc/ssl/certs/ca-certificates.crt` from the builder stage | Use `gcr.io/distroless/static-debian12:nonroot` (which includes CA certs) or add `COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/` |
| `go mod download` or `pip install` re-runs from scratch on every single code edit | `COPY . .` is placed before the dependency manifest copy (`COPY go.mod go.sum ./`), invalidating layer cache | Copy dependency lockfiles and run the package download step before `COPY . .` |
| Temporary GitHub Actions runners miss BuildKit cache on every PR run | Local `--mount=type=cache` directories live on the runner VM disk and are wiped after the job ends | Build with `docker buildx build --cache-from type=gha --cache-to type=gha,mode=max` in CI |
