---
title: "Standardising Docker Compose V2 Across Environments"
slug: "dockerising-and-standardising-compose"
date: "2026-09-20"
description: "Standardize Docker Compose V2 setups, split local overrides from base production topology, and block staging database bleed with fail-fast variable gates."
tags: ["docker", "devops", "containers", "tutorial"]
status: published
section: 7
topic: containers
prev_slug: install-docker-engine-on-debian
next_slug: optimizing-dockerfiles-with-multi-stage-builds
tested_on:
- Docker Engine 26.1
- Compose v2.27
- Debian 12
cover_image: https://spmahapatra.github.io/recordings/dockerising-and-standardising-compose/architecture.svg
platforms: [devto, medium, github]
published_at:
  devto: null
  medium: null
  github: https://spmahapatra.github.io/dockerising-and-standardising-compose
---

# Standardising Docker Compose V2 Across Environments

Local Docker Compose setups drift fast when every repository invents its own conventions. Without an explicit top-level `name:` declaration and strict variable checks, two projects cloned into folders named `app/` will overwrite each other's persistent database volumes, and a leftover shell export like `DATABASE_HOST=db-staging.internal.net` can silently route local integration tests into a live staging database over VPN.

This blueprint standardises multi-container applications around the **Compose V2 Specification** used in the companion repository [`spmahapatra/rick-morty-api-helm`](https://github.com/spmahapatra/rick-morty-api-helm), removing deprecated `version:` headers, splitting production service definitions from local developer bind-mounts, and enforcing fail-fast `${VAR:?error}` environment gates before any container starts.

> [!INFO]
> **What This Blueprint Delivers:**
> - **Deterministic Project Namespaces:** Top-level `name:` replaces deprecated `version: '3.8'` headers so container, network, and volume prefixes never collide across folders.
> - **Two-File Topology Split:** Production service topology stays locked in `docker-compose.yml` while local source bind-mounts live in `docker-compose.override.yml`.
> - **Fail-Fast Variable Gate:** `${DATABASE_URL:?error}` halts `docker compose up` immediately if required `.env` variables are unset.

## Architecture Overview

When you run `docker compose up`, Compose V2 resolves configuration across three repository files (`docker-compose.yml`, `docker-compose.override.yml`, and `.env`) through the Compose Spec schema and variable interpolation gate before issuing a single container creation call to the Docker Engine API:

![Standardising Docker Compose V2 Architecture Overview](https://spmahapatra.github.io/recordings/dockerising-and-standardising-compose/architecture.svg)

## Project Structure & Prerequisites

- **Docker Engine 24.0+:** Verify daemon health with `docker version --format '{{.Server.Version}}'`.
- **Docker Compose V2 (v2.20+):** Verify the Go plugin with `docker compose version` (Compose V1 `docker-compose` reached End-of-Life in July 2023).
- **Companion Reference Repository:** [`spmahapatra/rick-morty-api-helm`](https://github.com/spmahapatra/rick-morty-api-helm).

```plaintext
# file: Directory Layout
rick-morty-api-helm/
├── docker-compose.yml           # Locked production service topology + top-level name:
├── docker-compose.override.yml  # Local developer bind-mounts & debug ports (auto-merged)
├── .env.example                 # Committed template pointing to 127.0.0.1 dummy credentials
└── .env                         # Developer-local secrets (gitignored, never committed)
```

## Locking the Project Namespace (`docker-compose.yml`)

In Compose V1, top-level `version: '3.8'` headers pinned schema versions while project identity defaulted to the current directory name. Under the Compose V2 Specification, `version:` is deprecated and triggers runtime warnings. Audit your repository for legacy `version:` lines, remove them, and declare an explicit top-level `name:` key at line 1 of `docker-compose.yml`:

```bash
find . -maxdepth 3 -type f \( -name "docker-compose*.yml" -o -name "docker-compose*.yaml" \) \
  -exec grep -Hn "^version:" {} + || true
```

Define the base [`docker-compose.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.yml) with `name: core-platform-services` so every container, bridge network, and volume is deterministically prefixed regardless of where a developer clones the repository:

```yaml
# file: docker-compose.yml
name: core-platform-services

services:
  api:
    image: registry.internal.net/billing/app:v2.4.1
    restart: unless-stopped
    ports:
      - "${HOST_PORT:-8080}:8080"
    environment:
      - NODE_ENV=${NODE_ENV:-development}
      - DATABASE_URL=${DATABASE_URL:?DATABASE_URL must be explicitly set in .env}
    networks:
      - internal_net

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD:?REDIS_PASSWORD must be set in .env}
    networks:
      - internal_net

networks:
  internal_net:
    driver: bridge
```

> [!WARNING]
> **Caveats & Impact:** Changing the top-level `name:` creates a new volume and network prefix (`core-platform-services_*`). Always run `docker compose down --remove-orphans` before renaming a project so containers using the old folder-based prefix are stopped cleanly.

## Isolating Local Developer Mounts (`docker-compose.override.yml`)

Never pollute `docker-compose.yml` with host source bind-mounts (`volumes: - .:/usr/src/app`) or local debug targets. Keep local developer physics in [`docker-compose.override.yml`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/docker-compose.override.yml). Running `docker compose up` locally merges both files automatically, while CI/CD and staging jobs can target the locked production topology explicitly via `docker compose -f docker-compose.yml up -d`:

```yaml
# file: docker-compose.override.yml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: dev
    ports:
      - "5000:5000"
    volumes:
      - .:/usr/src/app
```

Verify that Compose V2 merges the base service definition with the local override mounts by inspecting the rendered canonical configuration:

```bash
DATABASE_URL="postgresql://local_user:local_pass@127.0.0.1:5432/app_db" \
REDIS_PASSWORD="local_dummy_redis_pass" \
docker compose config | head -n 22
```

```plaintext
name: core-platform-services
services:
  api:
    build:
      context: /home/localadmin/projects/rick-morty-api-helm
      dockerfile: Dockerfile
      target: dev
    environment:
      DATABASE_URL: postgresql://local_user:local_pass@127.0.0.1:5432/app_db
      NODE_ENV: development
    image: registry.internal.net/billing/app:v2.4.1
    networks:
      internal_net: null
    ports:
      - mode: ingress
        target: 8080
        published: "8080"
        protocol: tcp
      - mode: ingress
        target: 5000
        published: "5000"
        protocol: tcp
```

## Enforcing Fail-Fast Variable Gates (`.env.example`)

At 14:22 UTC during a past staging incident (`CRITICAL - DB_STAGING_MUTATION_ALERT`), a developer ran local destructive integration tests (`TRUNCATE TABLE users CASCADE;`) that silently connected to `db-staging.internal.net` over corporate VPN because their shell session still held an old exported variable and `docker-compose.yml` lacked a fail-fast `.env` contract.

Prevent cross-environment variable bleed by committing a sterile [`.env.example`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/.env.example) hardcoded to `127.0.0.1` dummy credentials and enforcing `${VAR:?error}` syntax on every sensitive connection string:

```bash
# file: .env.example
# Copy to .env before running docker compose up: cp .env.example .env
NODE_ENV=development
HOST_PORT=8080
DATABASE_URL=postgresql://local_dummy_user:local_dummy_pass@127.0.0.1:5432/local_dev_db
REDIS_PASSWORD=local_dummy_redis_pass
```

Verify that the fail-fast gate blocks startup immediately when `.env` is missing or when required variables are unset by running `docker compose config` in a stripped environment (`env -i`):

```bash
env -i HOME="$HOME" PATH="$PATH" docker compose config --quiet
```

```plaintext
error while interpolating services.api.environment.[]: required variable DATABASE_URL is missing a value: DATABASE_URL must be explicitly set in .env
```

> [!TIP]
> **Sanitized Invocation in Shared Shells:** Active shell `export` variables always override values inside `.env`. When switching between staging maintenance and local testing in the same terminal, launch Compose with `env -i HOME="$HOME" PATH="$PATH" docker compose up -d` so `.env` is the sole variable source.

## What This Setup Does Not Do & Troubleshooting

`docker compose config` validates YAML schema structure, file merging, and required variable presence, but it does not block an active shell `export` from taking precedence over a key defined in `.env`, nor does it encrypt secrets stored inside local `.env` files. Use this reference table to resolve common Compose V2 issues:

| Symptom / Error | Root Cause | Fix & Verify |
|---|---|---|
| Local container connects to staging host despite `127.0.0.1` in `.env` | Active shell `export DATABASE_URL=...` overrides `.env` file values in Compose variable precedence | Run `env -i HOME="$HOME" PATH="$PATH" docker compose config` to verify `.env` values without ambient shell exports |
| Repository B overwrites Postgres or Redis volume created by Repository A | Both repositories sit in folders named `app/` and omit the root `name:` key in `docker-compose.yml` | Add a unique top-level `name: <project-slug>` at line 1 of `docker-compose.yml` and verify with `docker volume ls` |
| CI pipeline fails with `docker-compose: command not found` | Standalone Python Compose V1 binary (`docker-compose`) reached EOL and was removed from modern runners | Replace hyphenated `docker-compose` commands with the Go CLI plugin `docker compose` (`docker compose version`) |
| `Warning: the attribute 'version' is obsolete, it will be ignored` | Legacy `version: '3.8'` key remains at top of `docker-compose.yml` or `docker-compose.override.yml` | Delete the `version:` line from all Compose files and run `docker compose config --quiet` |
