---
date: '2026-11-16'
description: Cut Flask REST API upstream latency from 320ms to 2ms using a deterministic
  Redis cache middleware with graceful fallback and LRU eviction.
platforms:
- devto
- medium
- github
published_at:
  devto: https://dev.to/spmahapatra/part-3-building-the-app-caching-for-data-latency-474i
  github: https://spmahapatra.github.io/caching-api-calls-efficiently
  medium: null
slug: caching-api-calls-efficiently
status: published
section: 7
topic: containers
prev_slug: optimizing-dockerfiles-with-multi-stage-builds
next_slug: ""
tested_on:
- Python 3.11 / Flask 3.0
- Redis 7.2 Alpine
- Docker Compose v2.27
cover_image: https://spmahapatra.github.io/recordings/caching-api-calls-efficiently/architecture.svg
tags:
- flask
- redis
- caching
- sre
title: Redis API Caching for Data Latency in Flask
---

# Redis API Caching for Data Latency in Flask

REST APIs that proxy or filter data from third-party upstream providers pay a 300ms+ public network penalty on every uncached `GET` request and fail immediately when the upstream provider rate-limits (`HTTP 429`) or drops connections. Even when teams add Redis, unsorted query parameters (`?status=alive&species=human` vs `?species=human&status=alive`) cause duplicate cache misses, and unguarded `redis.get()` calls crash the entire Flask worker whenever Redis restarts.

This blueprint implements the deterministic **Redis CacheMiddleware** from the companion repository [`spmahapatra/rick-morty-api-helm`](https://github.com/spmahapatra/rick-morty-api-helm), sorting query parameters before hashing keys, cutting repeat response times from **320ms (`X-Cache: MISS`) to 2ms (`X-Cache: HIT`)**, enforcing `allkeys-lru` memory bounds, and degrading gracefully to upstream fetches if Redis is unreachable.

> [!INFO]
> **What This Blueprint Delivers:**
> - **Deterministic MD5 Request Keys:** Canonicalizes `METHOD:endpoint:sorted_query_params` so identical queries always hit the same Redis key regardless of parameter order.
> - **2ms Cache-Aside Middleware (`X-Cache: HIT`):** Intercepts Flask `before_request` and `after_request` hooks with bounded TTL hydration.
> - **Fail-Open Resilience & LRU Eviction:** Wraps Redis I/O in `try/except RedisError` blocks and caps Redis memory with `maxmemory-policy allkeys-lru`.

## Architecture Overview

Every incoming `GET /api/v1/characters` request passes through `CacheMiddleware` before hitting the upstream HTTP client. On a cache hit, Flask returns the cached JSON payload from Redis in ~2ms with `X-Cache: HIT`. On a cache miss (or if Redis is down), Flask fetches from the upstream Rick & Morty API (~320ms), hydrates Redis with a bounded TTL (`300s`), and returns `X-Cache: MISS`:

![Redis API Caching for Data Latency in Flask Architecture Overview](https://spmahapatra.github.io/recordings/caching-api-calls-efficiently/architecture.svg)

## Project Structure & Prerequisites

- **Docker Engine 24.0+ & Compose V2 (v2.20+):** Used to run the Flask API and Redis 7 Alpine containers on a shared bridge network.
- **Python 3.11+ with `flask` and `redis`:** Pre-configured inside the companion repository container image.
- **Companion Reference Repository:** [`spmahapatra/rick-morty-api-helm`](https://github.com/spmahapatra/rick-morty-api-helm).

```plaintext
# file: Directory Layout
rick-morty-api-helm/
├── app.py                       # Flask REST entrypoint wiring CacheMiddleware
├── config.py                    # Redis host, socket timeout (0.5s) & TTL settings
├── docker-compose.yml           # Flask api:5000 + redis:6379 with allkeys-lru cap
└── src/
    └── cache/
        └── detection.py         # Deterministic key generator & fail-open middleware
```

## Canonicalizing Cache Keys (`src/cache/detection.py`)

If two clients request `/api/v1/characters?species=human&status=alive` and `/api/v1/characters?status=alive&species=human`, hashing the raw URL string creates two separate Redis keys for identical data. In [`src/cache/detection.py`](https://github.com/spmahapatra/rick-morty-api-helm/blob/main/src/cache/detection.py), sort query parameters alphabetically before generating the MD5 digest and wrap all Redis socket calls in short timeouts (`socket_timeout=0.5`) so a stalled cache never blocks Flask workers:

```python
# file: src/cache/detection.py
import hashlib
import json
import logging
from flask import Response, g, request
from redis.exceptions import RedisError

logger = logging.getLogger(__name__)


class CacheDetector:
    """Build deterministic cache keys across unordered HTTP query strings."""

    @staticmethod
    def generate_cache_key(endpoint: str, method: str, query_params: dict | None = None) -> str:
        key_base = f"{method.upper()}:{endpoint}"
        if query_params:
            sorted_qs = "&".join(f"{k}={v}" for k, v in sorted(query_params.items()))
            key_base = f"{key_base}:{sorted_qs}"
        digest = hashlib.md5(key_base.encode("utf-8")).hexdigest()
        return f"rm_api:v1:{digest}"


class CacheMiddleware:
    """Flask before/after request cache-aside middleware with fail-open fallback."""

    def __init__(self, app, redis_client, ttl_seconds: int = 300) -> None:
        self.redis = redis_client
        self.ttl = ttl_seconds
        self.detector = CacheDetector()
        app.before_request(self._check_cache)
        app.after_request(self._hydrate_cache)

    def _check_cache(self):
        if request.method != "GET" or request.headers.get("Cache-Control") == "no-cache":
            g.cache_status = "BYPASS"
            return None

        g.cache_key = self.detector.generate_cache_key(
            request.path, request.method, request.args.to_dict()
        )
        try:
            cached_payload = self.redis.get(g.cache_key)
            if cached_payload is not None:
                g.cache_status = "HIT"
                resp = Response(cached_payload, status=200, mimetype="application/json")
                resp.headers["X-Cache"] = "HIT"
                resp.headers["X-Cache-Key"] = g.cache_key
                return resp
        except RedisError as exc:
            logger.warning("Redis read failed for key=%s (%s); degrading to upstream", g.cache_key, exc)

        g.cache_status = "MISS"
        return None

    def _hydrate_cache(self, response: Response) -> Response:
        status = getattr(g, "cache_status", "MISS")
        response.headers.setdefault("X-Cache", status)
        if status == "MISS" and response.status_code == 200 and hasattr(g, "cache_key"):
            try:
                self.redis.setex(g.cache_key, self.ttl, response.get_data(as_text=True))
                response.headers["X-Cache-Key"] = g.cache_key
            except RedisError as exc:
                logger.warning("Redis write failed for key=%s (%s)", g.cache_key, exc)
        return response
```

> [!WARNING]
> **Caveats & Impact:** Never cache non-`200` error responses (`HTTP 500` or `HTTP 429`) with the standard 300-second TTL, or a brief 2-second upstream blip will serve cached errors to users for 5 full minutes.

## Wiring Flask & Redis LRU Limits (`app.py` & `compose.yml`)

Register `CacheMiddleware` on the Flask application in `app.py` with a 500ms Redis socket timeout (`socket_connect_timeout=0.5`, `socket_timeout=0.5`) so a dead Redis container fails open in half a second rather than hanging the request for 30 seconds:

```python
# file: app.py
import os
import redis
import requests
from flask import Flask, jsonify, request
from src.cache.detection import CacheMiddleware

app = Flask(__name__)

redis_client = redis.Redis(
    host=os.getenv("REDIS_HOST", "redis"),
    port=int(os.getenv("REDIS_PORT", "6379")),
    decode_responses=True,
    socket_connect_timeout=0.5,
    socket_timeout=0.5,
)

CacheMiddleware(app, redis_client, ttl_seconds=int(os.getenv("CACHE_TTL_SECONDS", "300")))


@app.get("/api/v1/characters")
def get_characters():
    species = request.args.get("species", "human")
    status = request.args.get("status", "alive")
    upstream = requests.get(
        "https://rickandmortyapi.com/api/character",
        params={"species": species, "status": status},
        timeout=5,
    )
    upstream.raise_for_status()
    results = [
        {"id": c["id"], "name": c["name"], "origin": c.get("origin", {}).get("name")}
        for c in upstream.json().get("results", [])
        if "Earth" in c.get("origin", {}).get("name", "")
    ]
    return jsonify({"count": len(results), "characters": results})
```

In `docker-compose.yml`, cap Redis memory at `128mb` with `--maxmemory-policy allkeys-lru` so Redis automatically evicts the least-recently-used character keys instead of crashing with an Out-Of-Memory (`OOM command not allowed`) error:

```yaml
# file: docker-compose.yml
name: rick-morty-platform

services:
  api:
    build: .
    ports:
      - "5000:5000"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - CACHE_TTL_SECONDS=300
    depends_on:
      - redis

  redis:
    image: redis:7.2-alpine
    container_name: redis-cache
    command: >
      redis-server
      --maxmemory 128mb
      --maxmemory-policy allkeys-lru
      --save ""
      --appendonly no
```

## Verifying MISS-to-HIT Latency & TTL (`curl` & `redis-cli`)

Start the Flask API and Redis cache containers in the background, then execute two consecutive `GET` requests with reversed query parameter ordering (`?species=human&status=alive` followed by `?status=alive&species=human`) to verify both the ~150x latency drop and canonical key reuse:

```bash
docker compose up -d

# First request (cold cache): fetches from upstream (~320ms, X-Cache: MISS)
curl -i -s "http://localhost:5000/api/v1/characters?species=human&status=alive" | grep -E "HTTP/|X-Cache"

# Second request (reversed query params): served from Redis (~2ms, X-Cache: HIT)
curl -i -s "http://localhost:5000/api/v1/characters?status=alive&species=human" | grep -E "HTTP/|X-Cache"
```

```plaintext
HTTP/1.1 200 OK
X-Cache: MISS
X-Cache-Key: rm_api:v1:6c8f3b219e4a1109d8b7451c20e4a912
HTTP/1.1 200 OK
X-Cache: HIT
X-Cache-Key: rm_api:v1:6c8f3b219e4a1109d8b7451c20e4a912
```

Inspect the live key TTL and memory eviction policy directly inside the `redis-cache` container:

```bash
docker exec redis-cache redis-cli TTL "rm_api:v1:6c8f3b219e4a1109d8b7451c20e4a912"  # Expected: (integer) 294
docker exec redis-cache redis-cli CONFIG GET maxmemory-policy                         # Expected: allkeys-lru
```

> [!TIP]
> **Bypassing Cache & Clean-Up / Revert:** Pass `-H "Cache-Control: no-cache"` on any `curl` test request to force `CacheMiddleware` to skip Redis (`X-Cache: BYPASS`) and test the live upstream API path directly. To flush cached keys or clean up and stop the local stack when finished, run `docker exec redis-cache redis-cli FLUSHALL && docker compose down -v`.

## What This Setup Does Not Do & Troubleshooting

This cache-aside middleware accelerates read-heavy `GET` endpoints and shields the app from transient Redis restarts, but it does not prevent a **Thundering Herd** cache stampede if 200 concurrent workers miss an expired key at the exact same millisecond, nor does it persist cached entries across Redis container restarts (`--save "" --appendonly no`). Use this table to diagnose caching anomalies:

| Symptom / Error | Root Cause | Fix & Verify |
|---|---|---|
| Identical queries with reordered parameters (`?a=1&b=2` vs `?b=2&a=1`) both return `X-Cache: MISS` | Cache key generator hashes `request.full_path` raw string without sorting query parameters | Sort `request.args.to_dict().items()` alphabetically in `generate_cache_key()` and verify both URLs return the same `X-Cache-Key` |
| Flask returns `HTTP 500` when the `redis-cache` container restarts | `redis.get()` or `redis.setex()` raises an unhandled `redis.exceptions.ConnectionError` or blocks on infinite socket timeout | Set `socket_timeout=0.5` on `redis.Redis()` and wrap `get`/`setex` calls in `try/except RedisError` |
| Redis logs `OOM command not allowed when used memory > 'maxmemory'` | Redis reached its memory limit with the default `noeviction` policy | Start `redis-server` with `--maxmemory 128mb --maxmemory-policy allkeys-lru` and verify via `redis-cli CONFIG GET maxmemory-policy` |
| Upstream API suffers a rate-limit spike every 300 seconds when a hot key expires | Hard TTL expiry causes concurrent threads to miss simultaneously (Thundering Herd) | Stagger TTLs with small random jitter (`300 + random.randint(0, 30)`) or refresh hot keys asynchronously via a background worker before hard expiry |
