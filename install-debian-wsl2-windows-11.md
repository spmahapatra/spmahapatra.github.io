---
date: '2026-05-04'
description: Set up a Debian WSL2 environment on Windows 11 with systemd, memory caps,
  and snapshot backups in under 15 minutes.
platforms:
- devto
- medium
- github
published_at:
  devto: https://dev.to/spmahapatra/how-to-set-up-wsl2-on-windows-11-53oi
  github: https://spmahapatra.github.io/install-debian-wsl2-windows-11
  medium: null
slug: install-debian-wsl2-windows-11
status: published
section: 7
topic: linux-wsl2
prev_slug: ""
next_slug: install-docker-engine-on-debian
tested_on:
- Windows 11 23H2
- WSL 2.2
- Debian 12
cover_image: https://spmahapatra.github.io/recordings/install-debian-wsl2-windows-11/architecture.svg
tags:
- wsl2
- debian
- windows
- devops
title: Installing Debian on WSL2 in Windows 11
---

# Installing Debian on WSL2 in Windows 11

Running Linux containers and system services on a Windows 11 workstation breaks quickly when WSL2 is installed from the Microsoft Store wrapper or when repositories are cloned under `/mnt/c/Users/`. The Store package ships a patched `/init` binary that silently blocks `systemd` units from starting, while cross-OS 9P filesystem mounts degrade `git` and `npm` I/O by 10x to 30x and allow the Hyper-V utility VM to consume host RAM without a cap.

This guide provisions a clean Debian 12 WSL2 instance directly from the official `wsl.exe` control plane, enables native `systemd` boot management, enforces host memory and mirrored networking bounds via `.wslconfig`, and exports a reproducible `.tar` base snapshot before project tooling is installed.

> [!INFO]
> **What This Blueprint Delivers:**
> - **Native systemd Boot Plane:** Direct `wsl --install -d Debian` provisioning with `/etc/wsl.conf` so `systemctl` manages services identically to bare-metal Debian.
> - **Host RAM Cap & Mirrored Networking:** Explicit `.wslconfig` bounds (`memory=4GB`, `networkingMode=mirrored`) so the Hyper-V VM never starves Windows 11.
> - **Fast ext4 Storage & Golden Snapshot:** All repositories live inside `~/projects` on `ext4.vhdx` backed by a clean `wsl --export` recovery archive.

## Architecture Overview

When you run `wsl -d Debian` on Windows 11, the Windows host boots a single shared Hyper-V utility virtual machine and mounts the Debian `ext4.vhdx` virtual disk inside it. Keeping active code inside the Linux `ext4` filesystem avoids the slow 9P translation boundary over `/mnt/c/`:

![Installing Debian on WSL2 Architecture Overview](https://spmahapatra.github.io/recordings/install-debian-wsl2-windows-11/architecture.svg)

## Project Structure & Prerequisites

- **Windows 11 Build 22621+ (22H2 / 23H2):** Verify your host build with `winver` in PowerShell.
- **Hardware Virtualization Enabled:** Confirm Intel VT-x or AMD-V is active under Task Manager → Performance → CPU (`Virtualization: Enabled`).
- **Administrator PowerShell:** Required for initial Hyper-V feature activation and `wsl --install`.
- **Disk Capacity:** At least 10 GB of free SSD storage on `C:\` for the virtual disk and clean `.tar` backup snapshot.

```plaintext
# file: Directory Layout
C:\Users\<WindowsUser>\
├── .wslconfig                         # Host-level Hyper-V RAM cap & mirrored network mode
└── wsl-backups\
    └── debian-clean.tar               # Golden pre-tooling snapshot (wsl --export)

\\wsl$\Debian\ (ext4.vhdx)
├── etc/
│   └── wsl.conf                       # Guest-level systemd=true and default user lock
└── home/<unix_user>/
    └── projects/                      # Native ext4 workspace (10-30x faster than /mnt/c)
```

## Provisioning the WSL2 Hypervisor & Debian Rootfs (`wsl.exe`)

Open PowerShell as Administrator and provision the WSL2 hypervisor platform and Debian root filesystem directly from the command line. Avoid installing Debian from the Microsoft Store UI, as old Store stubs can skip WSL2 default version registration.

```powershell
# file: PowerShell (Admin)
wsl --install -d Debian
```

If `wsl --install` reports that features are disabled on a fresh corporate image, enable both underlying Windows features explicitly and set version 2 as the locked default before rebooting:

```powershell
# file: PowerShell (Admin)
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
wsl --set-default-version 2
Restart-Computer
```

After rebooting, launch Debian from PowerShell (`wsl -d Debian`), create your UNIX username and password when prompted, and verify from PowerShell that the distribution is running on the WSL2 kernel rather than legacy WSL1 syscall translation:

```powershell
# file: PowerShell
wsl -l -v
```

```plaintext
  NAME      STATE           VERSION
* Debian    Running         2
```

> [!WARNING]
> **Caveats & Impact:** If `VERSION` prints `1`, convert the filesystem image immediately with `wsl --set-version Debian 2`. WSL1 lacks a real Linux kernel and cannot run `systemd`, Docker Engine, or eBPF tools.

## Enabling Native systemd Boot & Base Tooling (`/etc/wsl.conf`)

By default, WSL2 launches `/init` as PID 1 without starting the `systemd` service manager. Write `/etc/wsl.conf` inside your Debian shell to lock both `systemd=true` and your default non-root login account:

```ini
# file: /etc/wsl.conf
[boot]
systemd=true

[user]
default=localadmin
```

Apply this configuration from inside Debian, install the standard build and diagnostic packages required by downstream container and Kubernetes toolchains, and mask the three non-applicable hardware units (`multipathd`, `systemd-remount-fs`, and `apparmor`) so `systemctl` never reports `degraded` after restart:

```bash
sudo tee /etc/wsl.conf > /dev/null <<'EOF'
[boot]
systemd=true

[user]
default=localadmin
EOF

sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y \
  curl wget git build-essential \
  ca-certificates gnupg lsb-release \
  htop tmux jq unzip

sudo systemctl mask multipathd.service systemd-remount-fs.service apparmor.service
```

Shut down the utility VM from PowerShell (`wsl --shutdown`), reopen Debian (`wsl -d Debian`), and verify that `systemd` owns PID 1 and reports a healthy `running` state:

```bash
systemctl is-system-running  # Expected: running
systemctl --failed           # Expected: 0 loaded units listed
```

## Capping Hyper-V Memory & Mirrored Networking (`.wslconfig`)

The Linux kernel inside WSL2 caches filesystem pages aggressively in RAM, and Hyper-V does not automatically return all freed page cache to Windows 11. Without an explicit boundary, builds inside Debian can consume 8–12 GB of host RAM. Create `C:\Users\<YourWindowsUser>\.wslconfig` on the Windows host to cap VM resources and enable Windows 11 23H2 mirrored networking:

```ini
# file: C:\Users\localadmin\.wslconfig
[wsl2]
memory=4GB
processors=4
swap=2GB
networkingMode=mirrored
dnsTunneling=true
autoProxy=true
```

Restart the WSL2 VM from PowerShell (`wsl --shutdown`) and confirm inside Debian that the memory ceiling (`3.8Gi`) and native `ext4` project workspace are active:

```bash
mkdir -p ~/projects
free -h | awk 'NR==1 || NR==2 {print}'  # Expected: Mem total ~3.8Gi
df -Th ~/projects                       # Expected: Type ext4 mounted on /
```

> [!TIP]
> **Workspace Performance Rule:** Always clone Git repositories inside `~/projects` (`ext4` on `/dev/sdc`) and open them in Windows Explorer via `explorer.exe .` or `\\wsl$\Debian\home\localadmin\projects`. Never run `git clone`, `npm install`, or Docker bind-mounts against `/mnt/c/Users/...`.

## Exporting a Reproducible Golden Snapshot (`debian-clean.tar`)

Before installing Docker Engine, Minikube, or project dependencies, export a tarball of your verified base Debian image from PowerShell. Restoring a corrupted environment from this archive takes less than 45 seconds compared to a full OS reinstall and package upgrade cycle:

```powershell
# file: PowerShell
New-Item -ItemType Directory -Force -Path C:\wsl-backups
wsl --shutdown
wsl --export Debian C:\wsl-backups\debian-clean.tar
```

If you ever break your local package tree or kernel modules during an experiment, restore the clean snapshot with two commands:

```powershell
# file: PowerShell
wsl --unregister Debian
wsl --import Debian C:\WSL\Debian C:\wsl-backups\debian-clean.tar --version 2
wsl -d Debian -e whoami  # Expected: localadmin
```

## What This Setup Does Not Do & Troubleshooting

This configuration provides a single-node Debian 12 Linux kernel on a Windows 11 workstation; it does not auto-start background Linux services when Windows boots until the first `wsl.exe` session is invoked, and it does not replace dedicated bare-metal Linux servers for multi-node production traffic. Keep these operational failure modes and fixes in mind:

| Symptom / Error | Root Cause | Fix & Verify |
|---|---|---|
| `Error: 0x80370102` during `wsl --install` or first distro boot | Hardware virtualization (`Intel VT-x` / `AMD-V`) is disabled in BIOS or `VirtualMachinePlatform` Windows feature is off | Enable VT-x/AMD-V in UEFI BIOS, run `dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all`, reboot, and verify `wsl --status` |
| `systemctl status` reports `System is degraded` inside Debian | WSL2 utility VM lacks physical multipath disks and AppArmor kernel modules expected by default Debian units | Mask unsupported units with `sudo systemctl mask multipathd.service systemd-remount-fs.service apparmor.service` and verify `Failed: 0 units` |
| `git status` or `docker compose build` takes 20x longer than normal | Repository is stored on the Windows NTFS drive under `/mnt/c/Users/...` across the 9P translation bridge | Move the repository into `~/projects/` inside Debian and verify `df -Th .` reports `ext4` instead of `9p` |
| Restored `wsl --import` instance logs in as `root` instead of your user | `wsl --import` resets the registry default UID unless `/etc/wsl.conf` specifies `[user] default=<name>` | Verify `/etc/wsl.conf` contains `[user]` with `default=<username>`, run `wsl --shutdown`, and test `wsl -d Debian -e whoami` |
