---
title: "Installing Debian on WSL2 in Windows 11"
slug: "install-debian-wsl2-windows-11"
description: "Set up a Debian WSL2 environment on Windows 11 with systemd, backup/restore, and advanced config in under 15 minutes."
tags: [wsl2, debian, windows, devops]
status: ready
platforms: [devto, medium, github]
published_at:
  devto: null
  medium: null
  github: null
---

<!-- VOICE CHECKLIST — delete before publishing
     Description: no "production-grade / comprehensive / robust / powerful" — say what they'll have instead.
     Em dashes: max 3-4 per article. Replace extras with commas or periods.
     Repeated phrases: if a phrase appears in two sections, cut one.
     Summary sentences: delete any sentence that just restates the one before it.
     Opinion: add one sentence of honest preference, frustration, or opinion. AI can't fake this. -->

# Installing Debian on WSL2 in Windows 11

## Key Takeaways

- WSL2 with Debian gives you a full Linux kernel and systemd support — a legitimate CI parity environment on Windows hardware without running a separate VM.
- Take a snapshot with `wsl --export` before installing any project tooling (Step 7). It's a one-minute operation that has saved me multiple full reinstalls.
- Steps 1–5 get you a working Debian environment with systemd. Steps 6–7 (Windows Terminal, snapshot) are worth doing the same session.

## What Is WSL2?

**WSL2 (Windows Subsystem for Linux 2) ships a real Linux kernel inside a lightweight Hyper-V virtual machine.** Unlike WSL1, which translated Linux syscalls into Windows calls, WSL2 runs an actual Linux kernel — meaning full syscall compatibility, `systemd`, and `eBPF` all work without a separate VM.

As of Windows 11 22H2, WSL2 supports `systemd` natively with no workarounds, persistent background services, and GPU passthrough via CUDA on supported hardware. This makes it the first Windows-native environment where you can run production-parity workloads alongside your Windows desktop.

## The Mental Model

WSL2 runs inside a single lightweight Hyper-V VM (the "WSL2 utility VM"). Each installed distro is a separate filesystem image (`ext4.vhdx`) running inside that VM. The VM boots once; all distros share the kernel.

1. **The Windows filesystem (`C:\`) is mounted at `/mnt/c/`** — I/O across this boundary is slow. Keep project files in `~/` (inside the distro), not in `/mnt/c/Users/...`
2. **`wsl.exe` is your control plane**: install, unregister, export, import, and configure distros from PowerShell. Think of it as `docker` but for Linux distros.
3. **Networking is NAT'd by default**: WSL2 gets its own IP that changes on reboot. Windows 11 23H2+ adds a mirrored networking mode that eliminates this.

## Prerequisites

- Windows 11 (Build 22000 or later — verify with `winver`)
- PowerShell running as Administrator
- Virtualization enabled in BIOS (check Task Manager → Performance → CPU → "Virtualization: Enabled")
- 4 GB free disk space minimum; 10 GB recommended for a comfortable base install

## Walkthrough

### 1. Install WSL2 and Debian

Open PowerShell as Administrator (right-click Start → **Terminal (Admin)**).

**Fast path — works on most Windows 11 machines:**

```powershell
wsl --install -d Debian
```

This enables the WSL feature, sets version 2 as default, and installs Debian. Once the install finishes, a Debian terminal opens automatically and prompts you to create a user account:

```
Enter new UNIX username: yourname
New password:
Retype new password:
passwd: password updated successfully
```

The password won't display as you type — that's normal. Once done, close the Debian window and reboot:

```powershell
Restart-Computer
```

> **Warning:** The Microsoft Store's Debian package ships a patched `/init` that silently breaks `systemd` — services you enable will appear to succeed but never start. Use `wsl --install` above, not the Store app, if you need reliable service management.

**Manual path — use this if `wsl --install` errors out:**

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
wsl --set-default-version 2
```

Reboot to activate the hypervisor, then after restart run `wsl --install -d Debian` again — the username/password prompt will appear automatically when install completes.

### 2. Launch Debian After Reboot

After rebooting, Debian won't open automatically. Launch it from PowerShell:

```powershell
wsl -d Debian
```

You should land at your user prompt:

```
yourname@MACHINE-NAME:~$
```

**What to verify:** Launch it from PowerShell:

```powershell
wsl --status
# Default Version: 2
# Kernel version: 5.15.x or higher

wsl -l -v
# NAME      STATE     VERSION
# Debian    Running   2
```

If VERSION shows `1`, force the upgrade:

```powershell
wsl --set-version Debian 2
```

### 3. Launch Debian — Four Ways

After first-launch setup, Debian won't automatically open on future reboots. Pick whichever method fits your workflow:

**From PowerShell or CMD (fastest):**
```powershell
wsl -d Debian
```
Or just `wsl` if Debian is your default distro.

**From the Start menu:**
Search for **Debian** — it appears as an app. Pin it to your taskbar for one-click access.

**From Windows Terminal:**
Click the **`+`** dropdown tab arrow → select **Debian**. Each click opens a new tab in your existing terminal window.

**From any File Explorer folder:**
Right-click inside a folder → **Open Linux shell here** (Windows 11 with Terminal installed). Opens a Debian shell with that folder already set as the working directory — useful when you want to run Linux tools on Windows files.

> **Tip:** If you get dropped into a `root` shell instead of your user account, check that `[user] default=yourname` is set in `/etc/wsl.conf` (Step 4 covers this).

### 4. Enable systemd

Inside your Debian terminal:

```bash
sudo tee /etc/wsl.conf > /dev/null <<'EOF'
[boot]
systemd=true

[user]
default=your_unix_username
EOF
```

Restart WSL from PowerShell, then re-enter the distro:

```powershell
wsl --shutdown
wsl -d Debian
```

**What to verify:**

```bash
systemctl --no-pager status
# State: running
# (NOT "System is degraded" — if degraded, run: systemctl --failed)
```

### 5. Update and Install Base Packages

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y \
  curl wget git build-essential \
  ca-certificates gnupg lsb-release \
  htop tmux jq unzip

sudo timedatectl set-timezone America/New_York   # adjust to your zone
```

**What to verify:**

```bash
timedatectl
# Local time: ...
# System clock synchronized: yes
```

### 6. Install Windows Terminal

WSL2 works in the default console but Windows Terminal is significantly better — tabs, split panes, per-distro profiles, and proper font rendering for tools like `tmux` and `htop`.

```powershell
winget install Microsoft.WindowsTerminal
```

Or install from the Microsoft Store. Once installed, open it and your Debian distro will already appear as a profile in the `+` dropdown — no configuration needed.

**What to verify:**

Open Windows Terminal → click the `+` dropdown → confirm **Debian** appears as a profile. Select it — you should land at your Debian user prompt, not a root shell.

### 7. Export a Backup Snapshot Before Installing Anything Else

Before you install project-specific tooling, export a clean snapshot.

```powershell
# In PowerShell — create the backup directory first
New-Item -ItemType Directory -Force -Path C:\wsl-backups
wsl --export Debian C:\wsl-backups\debian-clean.tar
```

To restore from backup:

```powershell
wsl --unregister Debian
wsl --import Debian C:\WSL\Debian C:\wsl-backups\debian-clean.tar --version 2
```

**What to verify:**

```powershell
wsl -d Debian -e whoami
# Should print your username, not root
```

## Accessing Files Between Windows and Linux

This trips up almost every first-timer. WSL2 runs in its own filesystem, but both sides can reach each other.

**From inside Debian — access Windows files:**

```bash
ls /mnt/c/Users/YourWindowsUsername/
```

Your Windows drives are mounted under `/mnt/`.

> **Warning:** I/O across this boundary is 10–30× slower than native Linux I/O. Never run `npm install`, `git clone`, or heavy build operations on files under `/mnt/c/` — always keep project files in `~/` inside the distro.

**From Windows Explorer — browse your Linux filesystem:**

Type this path directly in the Explorer address bar:

```
\\wsl$\Debian
```

This opens your Debian home directory in Windows Explorer. You can drag, copy, and edit files here as if they were Windows files.

**Shortcut — open current Linux folder in Explorer:**

```bash
explorer.exe .
```

Run this from any directory inside WSL2 and it opens that exact folder in Windows Explorer. Useful for accessing build artifacts from your IDE.

## WSL Command Reference

| Command | What it does |
|---|---|
| `wsl --install -d Debian` | Install a specific distro |
| `wsl --list --online` | Show all available distros |
| `wsl -l -v` | List installed distros with version and state |
| `wsl --shutdown` | Stop all running WSL instances |
| `wsl --update` | Update the WSL kernel |
| `wsl -d Debian` | Launch a specific distro by name |
| `wsl --export Debian file.tar` | Export distro to a backup file |
| `wsl --import Debian path file.tar` | Restore distro from backup |
| `wsl --unregister Debian` | Remove a distro (destructive — backup first) |
| `wsl --set-version Debian 2` | Convert an existing distro to WSL2 |

## Advanced Configuration

These settings are optional — apply them once your base Debian environment is working.

### Capping WSL2 Memory Usage

The Hyper-V VM holds memory it has used even after processes exit. Without a cap it can consume several GB that Windows never reclaims. Create `C:\Users\YourName\.wslconfig`:

```ini
[wsl2]
memory=4GB
processors=4
swap=2GB
```

Apply with `wsl --shutdown`. Verify inside Debian: `free -h` should show the capped total.

### Mirrored Networking (Windows 11 23H2+)

By default, WSL2 uses NAT and gets a dynamic IP that changes on reboot. Mirrored mode gives WSL2 the same IP as your Windows host — services are reachable from your LAN without port-forwarding rules.

Add to `C:\Users\YourName\.wslconfig`:

```ini
[wsl2]
networkingMode=mirrored
```

Apply with `wsl --shutdown`. Verify inside Debian: `ip addr` should show your Windows LAN IP.

## Debugging & Common Pitfalls

### `Error: 0x80370102` — virtualization not enabled

This is the most common error on first install. Reboot into BIOS/UEFI and enable **Intel VT-x** (Intel CPUs) or **AMD-V** (AMD CPUs). It's usually under Advanced → CPU Configuration. Some corporate laptops have this locked by IT policy.

### Distro stuck at "Installing" with 0% progress after reboot

Two common causes:

1. **Virtual Machine Platform not enabled** — rerun Step 1 and reboot.
2. **Microsoft Store pending update** — open the Microsoft Store, check for pending updates, let them finish, then try again. The Debian app itself sometimes needs to complete a Store update before first launch.

Inspect the event log for the underlying error:

```powershell
Get-WinEvent -LogName Microsoft-Windows-Hyper-V-Guest-Drivers/Admin -MaxEvents 20
```

### systemctl reports "System is degraded"

Run `systemctl --failed` to see which units failed. Three common WSL2-specific culprits that are safe to mask:

```bash
sudo systemctl mask multipathd.service      # storage multipath — not applicable in WSL2
sudo systemctl mask systemd-remount-fs.service  # WSL2 filesystem quirk
sudo systemctl mask apparmor.service        # WSL2 kernel has no AppArmor modules
```

### `wsl --install` says WSL is already installed but nothing works

Run `wsl --update` to make sure the kernel is current, then `wsl --shutdown` and try again:

```powershell
wsl --update
wsl --shutdown
wsl -d Debian
```

If the distro still won't launch, check its status: `wsl -l -v`. If STATE shows "Stopped" and it won't start, export it for backup, unregister, and re-import from your backup snapshot (Step 7, Export a Backup Snapshot).

## FAQ

### Can I run WSL2 on Windows 10?

Yes — WSL2 is available on Windows 10 version 1903 (Build 18362) and later. However, native `systemd` support requires Windows 11 22H2 or Windows 10 Build 22000+. On older Windows 10 builds you need the `genie` or `distrod` workaround to get systemd running. Everything else in this guide works as written.

### My WSL2 distro is slow when accessing files under `/mnt/c/`. Is this normal?

That slowness is real and it doesn't go away. Cross-filesystem I/O crosses a virtual filesystem boundary — Linux talking to a Windows-hosted NTFS volume through a translation layer. Always keep active project files inside the Linux distro filesystem (`~/projects/`), not under `/mnt/c/Users/...`. The gap can be 10–30× on large file operations like `npm install` or `git clone`.

### How do I access my WSL2 Debian from another machine on my LAN?

By default, WSL2 uses NAT and is not directly accessible from the LAN. Three options: (1) Windows port forwarding — `netsh interface portproxy add v4tov4 listenport=8080 listenaddress=0.0.0.0 connectport=8080 connectaddress=$(wsl hostname -I)`. (2) Switch to mirrored networking in `.wslconfig` under `[wsl2]` — available on Windows 11 23H2+ and gives WSL2 the same IP as your Windows host. (3) Expose services via a reverse proxy like Caddy running inside the distro, bound to `0.0.0.0`.
