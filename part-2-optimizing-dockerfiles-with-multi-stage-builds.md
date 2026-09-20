---
title: "Part 2: Optimizing Dockerfiles with Multi-Stage Builds"
slug: "part-2-optimizing-dockerfiles-with-multi-stage-builds"
description: "Learn how to use Docker multi-stage builds to shrink your image sizes, speed up CI/CD pipelines, and improve your container security posture."
tags: ["docker", "devops", "tutorial", "performance"]
status: draft
platforms: [devto, medium, github]
published_at:
  devto: null
  medium: null
  github: null
---

# Part 2: Optimizing Dockerfiles with Multi-Stage Builds

## Key Takeaways

**Platform engineers:** Reduce base image sizes by 80% to 95% while enforcing zero-toolchain runtime environments across build pipelines.

**SREs on-call:** Eliminate `KubeletHasDiskPressure` node evictions and shorten deployment image pull times from minutes to seconds during autoscaling events.

**First-timers:** Start by separating your build-time SDKs from runtime binaries using named `FROM` statements before tuning BuildKit caching mounts.

---

## What Is Multi-Stage Docker Builds?

Multi-stage Docker builds allow you to use multiple `FROM` statements in a single `Dockerfile`. Each `FROM` instruction starts a new build stage with a fresh environment, letting you copy artifacts directly from one stage to another while dropping unnecessary compilers, source files, and temporary tooling.

As of Docker 23.0 and BuildKit v0.11, multi-stage builds execute stages in parallel when dependencies permit, skipping unused stages entirely. Without multi-stage builds, shipping a Go, Rust, or Java application requires either carrying compiler toolchains directly into your execution environment or writing fragile host-level wrapper scripts to clean up files before layer commit.

---

## The Mental Model

Why do single-stage Dockerfiles consistently inflate image size and expose unneeded system packages? Container images build as a series of read-only stackable layers. Deleting a tool in a later `RUN` command does not reduce the storage footprint of the finished image; the layer containing that tool remains baked into the image history.

Understanding where image bloat originates requires breaking down the container execution stack into distinct operational boundaries:

1. **Build Toolchain & Compiler Layer** — Compilers, C headers, git binaries, package managers (your build stage configuration).
2. **Intermediate Artifact Layer** — Compiled binaries, object files, dependency download caches (BuildKit engine / host engine storage).
3. **Runtime Dependency Layer** — Shared dynamic libraries (`libc`, `musl`), CA certificates, system users (your final stage configuration).
4. **Container Runtime Engine Layer** — Namespace isolation, cgroups, `overlay2` copy-on-write storage driver execution (host OS kernel, out of your hands).

You control layers 1, 2, and 3. Single-stage images force layers 1, 2, and 3 into the final layer stack. Multi-stage builds decouple layer 1 and layer 2 into throwaway stages, pulling only layer 3 into the final image definition.

---

## The Incident

At 03:14 UTC, two nodes in our primary Kubernetes cluster threw `KubeletHasDiskPressure` warnings. Within four minutes, three critical payment-gateway pods were evicted. Autoscaling attempted to launch replacement pods on surviving nodes, but the new pods stalled in `ContainerCreating` for 240 seconds. 

The monitoring dashboard displayed the following event logs across the cluster:

```text
03:18:12Z node-04 kubelet: WARNING: Disk usage on image filesystem is at 89%
03:19:01Z node-04 kubelet: ERROR: Failed to pull image "registry.internal/payment/api:v2.1.4": context deadline exceeded
03:19:05Z node-05 kubelet: INFO: Evicting pod payment-api-7db5c6c644-8x2ql due to disk pressure
```

Why did image pulling take four minutes per node pull? I checked the image manifest in our internal registry and found the `payment/api:v2.1.4` image measured 1.85 GB.

I spent the first hour convinced our private registry proxy was bandwidth-throttled by cloud egress rules. It was not. I inspected the image layers using `docker history` and discovered the build shipped the entire Go SDK, GCC compiler, standard header files, git repository history, and npm cache directly into runtime. 

```text
CREATED BY                                      SIZE
/bin/sh -c apt-get update && apt-get install…   642MB
/bin/sh -c go build -o /app/server .            480MB
COPY dir:a83b12... in /src                      720MB
```

Why were we shipping 1.84 GB of build infrastructure to serve a single 22 MB statically compiled binary? 

Honestly, I find Go's default static linking flags annoying because `-ldflags="-w -s"` should be standard in every pipeline template, but engineers keep omitting it. The team had relied on a single-stage `Dockerfile` based on `golang:1.21` without stripping debug symbols or separating the build toolchain from the execution image.

---

## Prerequisites

- Docker Engine 20.10.0+ or Docker Desktop 4.0+ with BuildKit enabled (`DOCKER_BUILDKIT=1`).
- Basic familiarity with Docker CLI flags and container layer concepts.
- `[VERIFY: docker buildx version]` returns BuildKit engine active.

---

## Walkthrough

### 1. Audit image bloat and layer inventory

Before rewriting your build definition, establish a baseline size and inspect which directives introduce unnecessary byte overhead.

```bash
# Build the legacy single-stage image
docker build -t app:legacy -f Dockerfile.legacy .

# Inspect total size
docker image ls app:legacy

# Analyze layer breakdown
docker history app:legacy --format "table {{.ID}}\t{{.Size}}\t{{.CreatedBy}}"
```

**What to verify:**

Confirm the total size and identify whether package managers (`apt`, `apk`), compilers (`gcc`, `go`), or source trees account for more than 50% of the total footprint.

```bash
docker inspect app:legacy --format='{{.Size}}'
```

### 2. Isolate compilation dependencies using build stages

Define a distinct compilation stage using a named `FROM` clause. Allocate all heavy dependencies, header packages, and build scripts exclusively to this phase.

Create a new `Dockerfile`:

```dockerfile
# Stage 1: Build workspace
FROM golang:1.22-bookworm AS builder

WORKDIR /build

# Copy dependency manifests first to leverage layer caching
COPY go.mod go.sum ./
RUN go mod download

# Copy source code and compile binary
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /build/bin/server .
```

**What to verify:**

Verify that building only the intermediate stage succeeds without creating runtime side effects.

```bash
docker build --target builder -t app:builder-only .
docker image ls app:builder-only
```

### 3. Extract minimal binary targets into runtime images

Add a second `FROM` instruction targeting a minimal base image like `debian:bookworm-slim`, `alpine`, or `gcr.io/distroless/static-debian12`. Copy only the compiled binary from the `builder` stage using `COPY --from`.

```dockerfile
# Stage 1: Build workspace
FROM golang:1.22-bookworm AS builder

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /build/bin/server .

# Stage 2: Minimal Execution Runtime
FROM gcr.io/distroless/static-debian12:nonroot

WORKDIR /app

# Copy executable from builder stage
COPY --from=builder /build/bin/server /app/server

# Expose port and configure non-root runtime environment
EXPOSE 8080
USER 65532:65532

ENTRYPOINT ["/app/server"]
```

**What to verify:**

Run the new multi-stage build and compare the final image size against `app:legacy`.

```bash
docker build -t app:optimized .
docker image ls | grep -E "app\s+(legacy|optimized)"
```

The output must show a dramatic reduction in size (for Go applications, usually dropping from ~1 GB to under 30 MB):

```text
REPOSITORY   TAG         IMAGE ID       CREATED          SIZE
app          legacy      d3f2a1b4c5e6   10 minutes ago   1.85GB
app          optimized   a1b2c3d4e5f6   2 minutes ago    28.4MB
```

### 4. Optimize BuildKit cache mounts across workflow stages

While multi-stage builds shrink image size, re-downloading package caches on every build slows pipeline execution speed. Use BuildKit `--mount=type=cache` options to persist dependency directories across stages without saving those directories into image layers.

Modify the `builder` stage in your `Dockerfile`:

```dockerfile
FROM golang:1.22-bookworm AS builder

WORKDIR /build

COPY go.mod go.sum ./

# Mount Go module cache and build cache directories
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /build/bin/server .

FROM gcr.io/distroless/static-debian12:nonroot
WORKDIR /app
COPY --from=builder /build/bin/server /app/server
USER 65532:65532
ENTRYPOINT ["/app/server"]
```

**Fast path — Build using BuildKit enabled inline:**

```bash
DOCKER_BUILDKIT=1 docker build -t app:optimized-cached .
```

**Manual path — build engine explicit export command if default context fails:**

```bash
docker buildx build --output type=docker -t app:optimized-cached .
```

**What to verify:**

Execute two sequential builds and check the time delta on the second run.

```bash
time docker build -t app:optimized-cached .
```

The second run should complete in under two seconds due to BuildKit cache target hits.

---

## Decision Framework

Selecting the right base image strategy for your final runtime stage introduces specific trade-offs between attack surface, debugging convenience, and POSIX compliance.

| Base Image Strategy | Final Image Footprint | Package Manager Available | Vulnerability Exposure (CVEs) | Debugging Capabilities | Primary Use Case |
|---|---|---|---|---|---|
| **Full OS Base** (`debian:bookworm`, `ubuntu:22.04`) | 100 MB - 800 MB | Yes (`apt`) | High (100+ standard packages) | High (shell, curl, gdb present) | Complex legacy applications needing dynamic C libraries |
| **Minimal OS Base** (`alpine:3.19`) | 5 MB - 15 MB | Yes (`apk`) | Low (fewer system packages) | Moderate (busybox shell present) | Applications requiring POSIX utilities but low disk usage |
| **Distroless** (`distroless/static-debian12`) | 2 MB - 20 MB | No | Minimal (no shell or package tools) | Low (requires ephemeral debug containers) | Security-critical services with static binaries |
| **Scratch** (`scratch`) | 0 MB (empty layer) | No | Zero OS-level CVEs | None (no shell or tools) | Purely static binaries (`Go`, `Rust`) with self-contained assets |

Increasing security by stripping out shells (`distroless`/`scratch`) means your operational debugging workflow changes: you cannot `docker exec -it container sh` into a failing instance. You must rely on telemetry, distributed tracing, or ephemeral debug sidecars instead.

---

## Command Reference

| Command | Operational Purpose |
|---|---|
| `docker build --target <stage_name> -t img .` | Build up to a specific intermediate stage for debugging or testing. |
| `docker buildx build --cache-from type=gha --cache-to type=gha,mode=max .` | Enable inline GitHub Actions caching for multi-stage BuildKit execution. |
| `docker history --no-trunc <image_id>` | Display full un-truncated layer commands to identify layer bloat source. |
| `docker builder prune --filter type=exec.cachemount` | Clear persistent BuildKit cache mounts from local storage. |
| `docker build --build-arg BUILDKIT_INLINE_CACHE=1 -t img .` | Embed BuildKit cache metadata directly inside exported registry images. |

---

## Debugging & Common Pitfalls

### Missing CA certificates in `scratch` base images

When porting a Go or Rust application from `alpine` or `debian` to `scratch`, outgoing HTTPS requests fail immediately upon startup.

```text
2024/03/15 10:22:41 Error fetching upstream API: Get "https://api.stripe.com/v1/charges": x509: certificate signed by unknown authority
```

The `scratch` image is entirely empty. It contains no root TLS certificates in `/etc/ssl/certs/ca-certificates.crt`.

Copy the certificate bundle from your builder stage into your final stage:

```dockerfile
FROM golang:1.22-bookworm AS builder
RUN apt-get update && apt-get install -y ca-certificates update-ca-certificates

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /build/bin/server /app/server
ENTRYPOINT ["/app/server"]
```

### Invalidated layer caching from incorrect directive order

Every `COPY` or `RUN` statement invalidates the cache for all subsequent lines if the source files change. Placing `COPY . .` at the top of a multi-stage `Dockerfile` causes `go mod download` or `npm install` to run on every line modification, ruining CI pipeline speed.

```text
# BAD: Invalidates package cache on any source file edit
COPY . .
RUN go mod download
```

Separate dependency declarations from source files:

```dockerfile
# GOOD: Only re-downloads dependencies when manifests change
COPY go.mod go.sum ./
RUN go mod download
COPY . .
```

### Dynamic C library resolution failure (`glibc` vs `musl`)

A binary compiled on a standard Linux builder stage fails to execute in an Alpine runtime stage.

```text
exec /app/server: no such file or directory
```

The path `/app/server` exists, but the dynamic linker loader specified inside the ELF binary header (typically `/lib64/ld-linux-x86-64.so.2` from `glibc`) is absent in Alpine (`musl`-based).

You have two choices to fix this execution error:

1. Compile a fully static binary in your build stage:
   ```dockerfile
   RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o /app/server .
   ```
2. Switch your builder or runtime image so both use identical standard libraries (e.g., compile on `golang:alpine` if deploying to `alpine`).

---

## References

1. Docker Documentation: [Use multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
2. Open Container Initiative: [OCI Image Format Specification v1.0.1](https://github.com/opencontainers/image-spec)
3. Google Container Tools: [Distroless Container Images Specification](https://github.com/GoogleContainerTools/distroless)

---

## FAQ

### Does BuildKit automatically skip unused build stages?

BuildKit analyzes the target stage specified in your build invocation and builds a dependency graph. If a stage in your `Dockerfile` is not referenced by the final stage or by a `COPY --from` instruction, BuildKit completely skips the execution of that stage's commands.

### How do I debug runtime errors when my target image lacks a shell?

You cannot attach a bash shell to `scratch` or `distroless` containers directly. In Kubernetes 1.23+, use `kubectl debug` to attach an ephemeral container containing diagnostic tools (`busybox` or `nicolaka/netshoot`) to the running pod's process namespace:

```bash
kubectl debug -it <pod-name> --image=cgr.dev/chainguard/busybox --target=<container-name>
```

For local Docker testing, build a dedicated debug image target by adding a temporary stage ending in `FROM alpine` or `FROM debian:slim`.

### Can I copy artifacts from an existing external Docker image instead of a local stage?

Pass the target image directly into the `--from` flag of a `COPY` instruction. You do not need to write a pre-stage `FROM` line to pull isolated assets from third-party tools.

```dockerfile
# Copy the official HashiCorp Vault binary directly into your runtime image
COPY --from=hashicorp/vault:1.15.2 /bin/vault /usr/local/bin/vault
```

### Will multi-stage builds slow down CI/CD pipelines due to cache hits failing across ephemeral runners?

Ephemeral CI runners start with fresh Docker daemon storage, causing local layer caches to miss on every pipeline run. Mitigate this by exporting inline BuildKit build caches directly to your container registry using `--cache-to=type=registry` and `--cache-from=type=registry` flags during the `docker buildx build` execution step.
