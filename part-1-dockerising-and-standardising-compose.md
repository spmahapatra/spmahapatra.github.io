---
title: "Part 1 Dockerising and Standardising Compose"
slug: "part-1-dockerising-and-standardising-compose"
description: "Learn to standardize Docker Compose setups, properly split environment data from architectural config, and stop cross-environment pollution in your codebase."
tags: ["docker", "devops", "engineering", "tutorial"]
status: draft
platforms: [devto, medium, github]
published_at:
  devto: null
  medium: null
  github: null
---

# Part 1: Dockerising and Standardising Compose

*Navigation: [Part 2: Caching API Calls Efficiently](./part-2-caching-api-calls-efficiently.md) →*

## Welcome and Series Overview

Welcome to the team! If you are a new developer joining us, this series is your definitive onboarding guide. We are taking a raw application and scaling it to enterprise standards.

Here is our published journey:
1. **Docker Compose Standardisation** (This post) — Enforcing strict local environments.
2. **App Creation & Caching** (See Part 2) — Building the Flask app and using Redis to solve data latency.
3. **Enterprise ELK Logging** (See Part 3) — Structured JSON logs for observability.
4. **Full-Cycle CI/CD** (See Part 4) — Shifting CI left to reduce costs.

---

## Key Takeaways

**Platform engineers:** Standardise Compose setups across your repositories. Use the Compose Specification schema. Remove the `version` key and add a `name` at the top level to stop cross-repo namespace clashes.

**SREs on-call:** Stop silent cross-environment pollution and database corruption. This usually happens when local containers inherit shell exports instead of strict `.env` fallbacks.

**First-timers:** Start by splitting your configuration into two distinct layers: Data vs. Architecture.
- **Data (`.env`)**: Injects raw strings and secrets (e.g. `DB_PASSWORD`). Keep this developer-managed and *never* commit it.
- **Architecture (`docker-compose.override.yml`)**: Overrides base container physics for local dev (mapping ports to your host, mounting live `./src` folders). Keep this repo-managed and always commit it to version control.
Keep the base [`docker-compose.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.yml) strictly for production-ready topologies.

---

## What Is Dockerising and Standardising Compose?

We often see teams using ad-hoc [`docker-compose.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.yml) files. Standardising means we stop doing this. We replace them with a unified format using Compose V2. Instead of relying on custom wrapper scripts or raw `docker run` commands, we agree on a strict structure for our services, volumes, and networks.

As of July 2023, Docker Compose V1 (written in Python) reached End-of-Life (EOL). We must now rely on Compose V2 (written in Go), which natively reads the Compose Specification. **Under this specification, the top-level `version` is deprecated, and project identity is set by the root `name` field, not the folder name.** If you skip this, you will face non-deterministic container naming and silent variable overrides across developer machines and CI/CD agents.

---

## The Mental Model

When a developer runs `docker compose up`, it doesn't just run a simple script. It parses and resolves your configuration through four layers.

1. **Repository Configuration** — `.env` files, shell exports (what you control)
2. **Compose Engine** — YAML parsing, variable expansion, inheritance (your local CLI)
3. **Docker Engine API** — Translates YAML into API calls (your host daemon)
4. **Host Kernel** — cgroup and network allocation (host OS)

If two developers run `docker compose up` on the same commit but get different results, where did it break? Not at the Engine or Kernel. It broke at the Repository Configuration layer. Unvalidated environment variables mutated the setup before a single API call reached the daemon.

---

## The Incident

At 14:22 UTC, I received a critical page: `CRITICAL - DB_STAGING_MUTATION_ALERT`. Staging database records were mutating, causing broken foreign keys.

I checked the staging database audit logs immediately.

```
2023-11-14 14:21:03 UTC [18402]: [3-1] user=app_dev,db=staging_db LOG:  STATEMENT:  TRUNCATE TABLE users CASCADE;
```

A local developer was running integration tests, thinking they were using a local database. Instead, their local app connected to our staging database over the corporate VPN and ran a destructive migration. 

Why did this happen? It was a variable cascade bleed. The developer's project relied on an implicit `.env` file without a fallback for `DATABASE_HOST`. Because the developer had previously exported `DATABASE_HOST=db-staging.internal.net` in their shell session, the local `docker compose up` silently inherited it.

The trade-off is clear: fixing this requires tight environment file rules and explicit boundaries for every local stack. I spent two days clearing doubts with developers who insisted "it worked fine on my machine", before I finally set up a CI linter to reject unstandardised Compose files.

---

## Prerequisites

- **Docker Engine**: Version 24.0.0 or higher. Precheck your host version using `docker version --format '{{.Server.Version}}'`.
- **Docker Compose**: Version 2.20.0 or higher (`docker compose version`).
- **Assumed Knowledge**: Familiarity with container networking and Linux processes.

---

## Walkthrough

### 1. Precheck legacy configurations and remove deprecated attributes

Inspect the repository for legacy Docker Compose formats. Older files use top-level `version` keys (like `version: '3.8'`). This ignores modern features.

Run this command to find all YAML files with the deprecated `version` attribute:

```bash
find . -maxdepth 3 -type f \( -name "docker-compose*.yml" -o -name "docker-compose*.yaml" \) \
  -exec grep -Hn "^version:" {} +
```

Remove the `version` field from all files. Replace it with an explicit `name` attribute in your primary file. This ensures container and volume names are properly scoped.

```yaml
# Standardised Compose Specification Format
name: core-platform-services

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "${HOST_PORT:-8080}:8080"
    environment:
      - NODE_ENV=${NODE_ENV:-development}
      - DATABASE_URL=${DATABASE_URL:?Doubts: DATABASE_URL must be explicitly provided}
    networks:
      - internal_net

networks:
  internal_net:
    driver: bridge
```

**Verification:**

Validate that Compose parses the file without throwing schema errors:

```bash
docker compose config --quiet && echo "Schema is valid."
```

---

### 2. Establish a multi-file split

To prevent local settings from leaking into staging, separate the base topology from environment overrides. Use [`docker-compose.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.yml) for shared definitions and [`docker-compose.override.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.override.yml) for developer-specific local mounts.

Create the base [`docker-compose.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.yml):

```yaml
name: billing-system

services:
  app:
    image: registry.internal.net/billing/app:v2.4.1
    restart: unless-stopped
    environment:
      - APP_PORT=3000
    networks:
      - app_bus
```

Create the local development override file [`docker-compose.override.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.override.yml). Docker Compose automatically merges this file when you run `docker compose up`:

```yaml
services:
  app:
    build:
      context: .
      target: dev
    ports:
      - "3000:3000"
    volumes:
      - .:/usr/src/app
```

**Verification:**

Verify how Compose merges these files locally:

```bash
docker compose config
```

Verify that the output contains both the base service definition and the local volume mounts.

---

### 3. Implement strict environment variable validation

Unset shell variables can cause dangerous fallbacks. Use standard Compose syntax to enforce required variables.

* Syntax `${VAR:-default}` uses `default` if `VAR` is empty.
* Syntax `${VAR:?error_message}` stops execution if `VAR` is empty.

First, create a sample [`.env.example`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/.env.example) file that developers can copy to `.env`. This file must point to local dummy credentials, ensuring they never connect to staging by mistake.

```bash
# .env.example
# Copy this file to .env before running docker compose up

# Application Environment
NODE_ENV=development

# Database Connection (Hardcoded to local dummy container, NOT staging)
DATABASE_HOST=127.0.0.1
DATABASE_PORT=5432
DATABASE_USER=local_dummy_user
DATABASE_PASSWORD=local_dummy_pass
```

Next, enforce required variable checks inside your services block in [`docker-compose.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.yml):

```yaml
services:
  worker:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD:?Fatal error: REDIS_PASSWORD is missing.}
```

**Verification:**

Test that Compose throws an explicit error when executing without required variables:

```bash
env -i docker compose config
```

---

## Real-World Example

You can see a complete, working implementation of these standards in our [Rick & Morty API repository](https://github.com/spmahapatra/rick-morty-api-helm). 
Check out the split between [`docker-compose.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.yml), [`docker-compose.override.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.override.yml), and [`.env.example`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/.env.example) to see how we safely expose local ports without compromising the base production configuration.

---

## Debugging & Common Pitfalls

### Silent Environment Variable Overrides

**Symptom:**  
Services connect to unintended host ports or external services, bypassing `.env` values.

**Root Cause:**  
Compose resolves variables using this precedence:
1. Shell variables set in the active terminal
2. Environment variables set in `.env`
3. Default values in the [`docker-compose.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.yml)

If a developer exports `PORT=9000` in their shell, Compose will use `9000`, silently ignoring `PORT=3000` inside `.env`.

**Fix:**  
Clear ambient environment variables before invoking Compose:

```bash
env -i HOME="$HOME" PATH="$PATH" docker compose up -d
```

---

### Volume Namespace Bleed Across Repositories

**Symptom:**  
Spinning up an application in Repository B overwrites persistent data in database volumes created by Repository A.

**Root Cause:**  
Both repositories define generic volume keys like `volumes: db_data:` without setting a root `name` attribute. Compose defaults to the current parent directory name. If both projects are inside directories named `app/`, they will share the exact same volume namespace.

**Fix:**  
Explicitly assign a unique top-level `name` in every repository's [`docker-compose.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.yml).

---

## FAQ

### Why should I delete the top-level `version` line?

The `version` field enforced legacy schema constraints (like `'3.8'`). Modern Compose V2 automatically detects capabilities based on the target Docker Engine. Including a `version` string now triggers deprecation warnings and can cause Compose to ignore modern attributes.

### Can I run `docker-compose` (with a hyphen) in modern CI pipelines?

You should not rely on `docker-compose` anymore. The standalone Python binary reached End-of-Life in July 2023. Modern pipelines must use `docker compose` (space separated), which uses the Go-based plugin integrated into the Docker CLI.

### Final Thoughts on Container Architecture

Adopting these standards isn't just about appeasing the platform engineering team—it's about fundamentally improving developer velocity. By separating the environment data from the architectural configuration, we establish a robust contract that prevents the classic "it works on my machine" syndrome. When every developer, CI/CD runner, and production node speaks the exact same container language, you stop firefighting environment pollution and start shipping features. Stick to this discipline, and your infrastructure will scale elegantly alongside your team.
