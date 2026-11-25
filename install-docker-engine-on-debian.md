---
date: '2026-08-23'
description: Install Docker Engine on Debian from Docker's official signed APT repository,
  verify the daemon, and configure non-root socket access safely.
platforms:
- devto
- medium
- github
published_at:
  devto: https://dev.to/spmahapatra/install-docker-engine-on-debian-4dga
  github: https://spmahapatra.github.io/install-docker-engine-on-debian
  medium: null
slug: install-docker-engine-on-debian
status: published
section: 7
topic: containers
prev_slug: install-debian-wsl2-windows-11
next_slug: part-1-dockerising-and-standardising-compose
tested_on:
- Debian 12 (Bookworm)
- Docker Engine 26.1
- Compose v2.27
cover_image: https://spmahapatra.github.io/recordings/install-docker-engine-on-debian/architecture.svg
tags:
- docker
- debian
- containers
- linux
title: Install Docker Engine on Debian
---

# Install Docker Engine on Debian

Installing Docker on Debian by running `apt install docker.io` or mixing old `.list` repository files with modern `.sources` manifests causes two painful operational failures: outdated distribution runtime packages that lack the Go-based `docker compose` V2 plugin, and APT `Conflicting values set for option Signed-By` errors that halt system upgrades. Even when the daemon starts, skipping non-root socket verification causes downstream tools like Minikube to fail with misleading driver errors because the current login shell cannot open `/var/run/docker.sock`.

This blueprint removes conflicting distribution packages, configures Docker's official deb822 signed APT repository (`/etc/apt/sources.list.d/docker.sources`), installs `docker-ce` alongside `containerd.io`, `docker-buildx-plugin`, and `docker-compose-plugin`, and verifies non-root daemon execution on Debian 11, 12, and 13.

> [!INFO]
> **What This Blueprint Delivers:**
> - **Clean deb822 Signed APT Source:** Uses `/etc/apt/keyrings/docker.asc` and `/etc/apt/sources.list.d/docker.sources` without conflicting legacy `.list` files.
> - **Complete Official Runtime Stack:** Installs `docker-ce`, `docker-ce-cli`, `containerd.io`, Buildx, and Compose V2 from a single verified release stream.
> - **Verified Non-Root Socket Access:** Validates `/var/run/docker.sock` group permissions and container execution in the exact shell used by downstream tools.

## Architecture Overview

Installing Docker Engine on Debian separates package trust from runtime execution. APT verifies packages against `/etc/apt/keyrings/docker.asc` before installing the `dockerd` and `containerd` systemd services, while the unprivileged user CLI communicates with the daemon across `/var/run/docker.sock`:

![Install Docker Engine on Debian Architecture Overview](https://spmahapatra.github.io/recordings/install-docker-engine-on-debian/architecture.svg)

## Project Structure & Prerequisites

- **Debian 11 (Bullseye), 12 (Bookworm), or 13 (Trixie):** 64-bit (`amd64` or `arm64`) with `systemd` active as PID 1.
- **Sudo Privileges:** A non-root administrative account with `sudo` access.
- **Outbound HTTPS (443):** Access to Debian mirrors and `https://download.docker.com/linux/debian`.
- **Clean Firewall Policy:** Awareness that published container ports (`-p`) program `iptables-nft` rules directly.

```plaintext
# file: Directory Layout
/
├── etc/
│   ├── apt/
│   │   ├── keyrings/
│   │   │   └── docker.asc             # Official Docker GPG signing key (0644)
│   │   └── sources.list.d/
│   │       └── docker.sources         # deb822 repository manifest locked to codename
│   └── docker/
│       └── daemon.json                # Optional log-driver & storage configuration
└── var/
    ├── lib/docker/                    # Persistent image layers, volumes & containers
    └── run/docker.sock                # root:docker (0660) Unix domain control socket
```

## Purging Unofficial Packages & Legacy APT Lists (`apt remove`)

Debian's default repositories ship older `docker.io`, `podman-docker`, and standalone `containerd` packages that clash with Docker's official runtime bundle. Before configuring the repository, inspect your Debian release codename, purge any unofficial packages, and delete stale `/etc/apt/sources.list.d/docker.list` files so APT never encounters duplicate `Signed-By` definitions:

```bash
. /etc/os-release && echo "$VERSION_CODENAME ($(dpkg --print-architecture))"  # Expected: bookworm (amd64)

sudo apt-get remove -y docker.io docker-compose docker-doc docker-buildx podman-docker containerd runc || true
sudo rm -f /etc/apt/sources.list.d/docker.list
```

> [!TIP]
> **Data Preservation:** Running `apt-get remove` removes conflicting binaries only; it leaves any existing images, containers, and volumes under `/var/lib/docker` intact.

## Configuring the Signed deb822 Repository (`docker.sources`)

Create `/etc/apt/keyrings` with `0755` permissions, download Docker's official ASCII-armored GPG key to `/etc/apt/keyrings/docker.asc`, and write a deb822 `/etc/apt/sources.list.d/docker.sources` manifest pinned to your exact Debian codename and CPU architecture:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt-get update
```

Inspect `/etc/apt/sources.list.d/docker.sources` and confirm `apt-cache policy docker-ce` resolves packages from `download.docker.com`:

```deb822
# file: /etc/apt/sources.list.d/docker.sources
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: bookworm
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
```

```bash
apt-cache policy docker-ce
```

```plaintext
docker-ce:
  Installed: (none)
  Candidate: 5:26.1.4-1~debian.12~bookworm
  Version table:
     5:26.1.4-1~debian.12~bookworm 500
        500 https://download.docker.com/linux/debian bookworm/stable amd64 Packages
```

## Installing Docker Engine, Buildx & Compose V2 (`docker-ce`)

Install the complete official runtime bundle (`docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, and `docker-compose-plugin`), enable the `docker.service` systemd unit, and verify the daemon and CLI plugins:

```bash
sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin

sudo systemctl enable --now docker
sudo systemctl is-active docker          # Expected: active
sudo docker run --rm hello-world         # Expected: Hello from Docker!
docker compose version                   # Expected: Docker Compose version v2.27.1
docker buildx version                    # Expected: github.com/docker/buildx v0.14.1
```

## Granting Non-Root Socket Access (`/var/run/docker.sock`)

By default, `/var/run/docker.sock` is owned by `root:docker` with `0660` permissions. To run `docker`, `docker compose`, and `minikube --driver=docker` without prefixing every command with `sudo`, add your administrative user to the `docker` group, activate the new group membership, and verify unprivileged container execution:

```bash
sudo usermod -aG docker "$USER"
newgrp docker

ls -l /var/run/docker.sock               # Expected: srw-rw---- 1 root docker
id -nG                                   # Expected: localadmin sudo docker
docker run --rm hello-world              # Expected: Hello from Docker! (without sudo)
```

> [!WARNING]
> **Caveats & Impact:** Membership in the `docker` group grants root-equivalent privileges on the host because any user who can talk to `/var/run/docker.sock` can bind-mount `/` into a privileged container. Add only trusted administrators to `docker`, and always start a fresh login shell (or run `newgrp docker`) after `usermod` before launching Minikube or CI scripts.

## What This Setup Does Not Do & Troubleshooting

Installing Docker Engine does not configure automatic container log rotation (`max-size` in `/etc/docker/daemon.json`), nor does it prevent containers published with `-p 8080:8080` from bypassing `ufw` host firewall rules by inserting directly into `iptables`. Keep these common operational errors and fixes in mind:

| Symptom / Error | Root Cause | Fix & Verify |
|---|---|---|
| `permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock` | Active shell session has not loaded the new `docker` group after `usermod -aG docker "$USER"` | Log out and back in (or run `newgrp docker`), verify `id -nG` lists `docker`, and re-run `docker version` |
| `E: Conflicting values set for option Signed-By regarding source https://download.docker.com/linux/debian` | Both a legacy `/etc/apt/sources.list.d/docker.list` and new `docker.sources` exist with different key paths | Run `sudo rm -f /etc/apt/sources.list.d/docker.list`, verify only `docker.sources` remains, and run `sudo apt-get update` |
| `E: Unable to locate package docker-ce` on Debian derivative or testing release | `VERSION_CODENAME` in `/etc/os-release` does not match an official Docker Debian suite (`bullseye`, `bookworm`, `trixie`) | Edit `Suites:` in `/etc/apt/sources.list.d/docker.sources` to the matching upstream Debian codename and run `sudo apt-get update` |
| Port published via `-p 8080:8080` is reachable from external LAN despite `ufw deny 8080` | Docker Engine programs `iptables` NAT chains directly ahead of `ufw` input chains | Bind local services explicitly to loopback (`-p 127.0.0.1:8080:8080`) or add filtering rules to the `DOCKER-USER` iptables chain |
