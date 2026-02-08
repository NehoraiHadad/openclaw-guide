# Platform Reference: WSL2 (Windows Subsystem for Linux 2)

## Overview

OpenClaw on Windows runs inside WSL2, not natively on Windows. The CLI, gateway, and all integrations execute within a Linux distribution (Ubuntu recommended). This provides runtime consistency with Linux servers while running on a Windows desktop.

## Prerequisites

### Windows Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| Windows version | Windows 10 build 19041+ | Windows 11 22H2+ |
| WSL version | WSL 2 | WSL 2 (latest via `wsl --update`) |
| Distro | Any Linux distro | Ubuntu 24.04 LTS |
| RAM allocated | 4 GB | 8 GB+ (see `.wslconfig`) |

### Installing WSL2

From PowerShell (Administrator):

```powershell
# Install WSL2 with Ubuntu (default)
wsl --install

# Or specify a distro explicitly
wsl --install -d Ubuntu-24.04
```

After installation, set up your Linux user account in the new terminal window.

### Resource Limits (Optional)

Create `C:\Users\<you>\.wslconfig` to control memory/CPU allocation:

```ini
[wsl2]
memory=8GB
processors=8
swap=4GB
localhostForwarding=true
```

Apply changes with `wsl --shutdown` from PowerShell, then reopen your distro.

---

## Systemd Enablement

### Why It Matters

OpenClaw's gateway can run as a **systemd user service** for automatic startup, crash recovery, and clean lifecycle management. Without systemd, you must start the gateway manually in every session.

### Enabling systemd

Edit `/etc/wsl.conf` inside the WSL distro:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true

[user]
default=<your-username>
EOF
```

Then from PowerShell (not inside WSL):

```powershell
wsl --shutdown
```

Reopen your distro and verify:

```bash
systemctl --user status
```

If you see active service listings, systemd is working. If you see `Failed to connect to bus`, systemd is not enabled.

### Gateway With systemd

```bash
# Install the service
openclaw gateway install
# or during initial setup:
openclaw onboard --install-daemon

# Manage the service
systemctl --user enable --now openclaw-gateway.service
systemctl --user status openclaw-gateway.service
systemctl --user restart openclaw-gateway.service
journalctl --user -u openclaw-gateway.service -f
```

### Gateway Without systemd

If systemd is unavailable or broken, run the gateway manually:

```bash
# Foreground (useful for debugging)
openclaw gateway start --verbose

# Background via nohup
nohup openclaw gateway start > ~/openclaw-gateway.log 2>&1 &

# Or use tmux/screen
tmux new-session -d -s openclaw 'openclaw gateway start'
```

**Downside:** No auto-restart on crash, no automatic startup on WSL boot.

---

## Installation Steps (WSL2-Specific)

### Method 1: Homebrew / Linuxbrew (Current Setup)

```bash
# Install Homebrew for Linux
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Add to PATH (follow post-install instructions)
eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"

# Install OpenClaw
brew install openclaw

# Verify
which openclaw
# Expected: /home/linuxbrew/.linuxbrew/bin/openclaw
```

### Method 2: npm Global Install

```bash
# Ensure Node.js 22+ is installed
node -v

# Install globally
npm i -g openclaw@latest

# Verify
which openclaw
```

### Method 3: NodeSource + npm (Fresh System)

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
npm i -g openclaw@latest
```

### Post-Install

```bash
openclaw onboard              # Interactive setup wizard
openclaw onboard --install-daemon  # Also install gateway service
openclaw doctor               # Verify health
```

---

## PATH Issues with NVM

This is the **most common WSL2 problem**. NVM (Node Version Manager) manages Node installations under `~/.nvm/versions/node/`, but the PATH it configures is only available in interactive shells. Systemd services, cron jobs, and non-login shells do not source `~/.bashrc` / `~/.zshrc`, so they cannot find `node` or `openclaw`.

### Symptoms

- `openclaw doctor` reports: **"Gateway service PATH missing required dirs: /home/nehor/.nvm/current/bin"**
- Gateway service fails to start via systemd
- `openclaw` works in your terminal but not in background services

### Diagnosis

```bash
# Check if NVM is the issue
which node
# NVM path: ~/.nvm/versions/node/v22.x.x/bin/node
# Brew path: /home/linuxbrew/.linuxbrew/bin/node (no NVM issue)

# Check the service file PATH
grep "PATH=" ~/.config/systemd/user/openclaw-gateway.service
```

### Solutions

**Option A: Symlink approach (quick fix)**

```bash
# Create a stable symlink NVM expects
ln -sf ~/.nvm/versions/node/$(nvm current) ~/.nvm/current
```

**Option B: Rebuild service file**

```bash
# Regenerate the service with correct paths
openclaw doctor --fix
# or
openclaw gateway install
```

**Option C: Avoid NVM for the service binary**

If OpenClaw is installed via Homebrew (as in this system), the service file should reference the Homebrew node path (`/home/linuxbrew/.linuxbrew/bin/node`) rather than NVM. The `doctor --fix` command can update this.

**Option D: Add NVM paths to the service environment**

Manually edit `~/.config/systemd/user/openclaw-gateway.service` and add the NVM bin directory to the `PATH` environment line, then reload:

```bash
systemctl --user daemon-reload
systemctl --user restart openclaw-gateway.service
```

---

## Service File Reference

The gateway systemd service file lives at:

```
~/.config/systemd/user/openclaw-gateway.service
```

Standard structure:

```ini
[Unit]
Description=OpenClaw Gateway (v2026.x.x)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart="/path/to/node" "/path/to/openclaw/dist/index.js" gateway --port 18789
Restart=always
RestartSec=5
KillMode=process
Environment=HOME=/home/<user>
Environment="PATH=/home/linuxbrew/.linuxbrew/bin:..."
Environment=OPENCLAW_GATEWAY_PORT=18789
Environment=OPENCLAW_GATEWAY_TOKEN=<token>
Environment="OPENCLAW_SYSTEMD_UNIT=openclaw-gateway.service"
Environment=OPENCLAW_SERVICE_MARKER=openclaw
Environment=OPENCLAW_SERVICE_KIND=gateway

[Install]
WantedBy=default.target
```

Key points:
- The `PATH` must include all directories needed to resolve `node` and any tools
- The `OPENCLAW_GATEWAY_TOKEN` in the service file should match the token in `~/.openclaw/openclaw.json`
- After editing, run `systemctl --user daemon-reload`

---

## Networking Considerations

### Localhost Access

By default, Windows can reach WSL2 services via `localhost` (when `localhostForwarding=true` in `.wslconfig`, which is the default). The OpenClaw dashboard at `http://127.0.0.1:18789/` is accessible from both WSL and the Windows browser.

### WSL2 Internal IP

WSL2 runs on a virtual network with a dynamic IP that changes on restart:

```bash
hostname -I
# Example: 172.24.176.163 100.67.48.68
```

The `172.x.x.x` address is the WSL2 NAT interface. Do not hardcode it.

### LAN Access (Exposing to Other Machines)

WSL2 services are not directly reachable from LAN. To expose the gateway to other machines on your network, set up port forwarding from PowerShell (Administrator):

```powershell
# Get the WSL IP
$WslIp = (wsl -d Ubuntu-24.04 -- hostname -I).Trim().Split(" ")[0]

# Forward Windows port to WSL
netsh interface portproxy add v4tov4 `
  listenaddress=0.0.0.0 listenport=18789 `
  connectaddress=$WslIp connectport=18789

# Allow through firewall
New-NetFirewallRule -DisplayName "OpenClaw Gateway" `
  -Direction Inbound -Protocol TCP -LocalPort 18789 -Action Allow
```

**Important:** The WSL2 IP changes after every `wsl --shutdown` or reboot. You must refresh the port forwarding rule each time.

### Tailscale (Recommended for Remote Access)

Tailscale runs natively inside WSL2 and provides a stable IP that survives restarts:

```bash
# Install Tailscale in WSL
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Your Tailscale IP is stable
tailscale ip -4
```

When `gateway.bind` is set to `"tailnet"` in the OpenClaw config, remote nodes and mobile devices can reach the gateway via the Tailscale IP without port forwarding.

---

## Known Limitations on WSL2

| Limitation | Details | Workaround |
|------------|---------|------------|
| No launchd | macOS service manager not available | Use systemd (see above) |
| Audio/voice | No native audio device in WSL2 | Voice/TTS features limited; use Windows-side audio apps |
| GPU access | Limited CUDA/GPU passthrough | Sufficient for OpenClaw (no GPU needed) |
| Filesystem perf | `/mnt/c/` (Windows FS) is slow | Keep OpenClaw data in Linux FS (`~/`) |
| USB devices | No direct USB passthrough (default) | Use `usbipd-win` for USB devices if needed |
| Dynamic IP | WSL2 IP changes on restart | Use Tailscale or refresh port forwarding |
| Systemd quirks | User services may not start on WSL boot | Enable `systemd=true`; may need `loginctl enable-linger <user>` |
| No GUI (default) | No desktop environment | Dashboard accessible via Windows browser at localhost |
| Disk space | Shares disk with Windows | Monitor with `df -h /` |
| Sleep/hibernate | WSL2 may be suspended by Windows | Disable Windows sleep or use `wsl --shutdown` scheduling |

---

## Performance Tips

### Keep Data on the Linux Filesystem

Files under `/mnt/c/` (the Windows mount) have 5-10x slower I/O than the native Linux filesystem. Always install OpenClaw and keep its data directories under the Linux home:

```
~/.openclaw/          # Config and state
~/.config/systemd/    # Service files
~/workspace/          # Agent workspace
```

Avoid: `/mnt/c/Users/.../openclaw/`

### Memory Tuning

If OpenClaw + Node.js uses too much memory, constrain WSL2 via `C:\Users\<you>\.wslconfig`:

```ini
[wsl2]
memory=8GB
swap=4GB
```

### Reduce WSL Idle Overhead

Windows 11 can auto-terminate idle WSL instances. To keep the gateway running, either:
- Disable idle timeout: add `vmIdleTimeout=-1` to `.wslconfig`
- Use a keep-alive (the gateway's heartbeat usually suffices)

### Node.js Flags

For large session stores or many concurrent agents:

```bash
# In the service file Environment or CLI
NODE_OPTIONS="--max-old-space-size=2048"
```

---

## Diagnostics Quick Reference

| Command | Purpose |
|---------|---------|
| `openclaw doctor` | Full health check with recommendations |
| `openclaw doctor --fix` | Auto-repair detected issues |
| `openclaw gateway status` | Gateway runtime, probe, and service status |
| `openclaw status` | Overall OpenClaw status |
| `openclaw logs --follow` | Live gateway log stream |
| `systemctl --user status openclaw-gateway.service` | Systemd service state |
| `journalctl --user -u openclaw-gateway.service -f` | Systemd service logs |
| `wsl --status` | WSL version and kernel info (from PowerShell) |
| `cat /etc/wsl.conf` | WSL boot and systemd config |

---

## Current System Details (nehor)

Captured via local inspection on 2026-02-08:

| Property | Value |
|----------|-------|
| WSL kernel | `6.6.87.2-microsoft-standard-WSL2` |
| Distro | Ubuntu 24.04.3 LTS (noble) |
| OpenClaw version | `2026.2.3-1` |
| Node.js version | `v25.5.0` |
| NVM version | `0.40.1` (installed but using system node) |
| NVM current | `system` (NVM default alias: v22.22.0) |
| Install method | Homebrew (Linuxbrew 5.0.13) |
| Binary path | `/home/linuxbrew/.linuxbrew/bin/openclaw` |
| Node binary | `/home/linuxbrew/.linuxbrew/bin/node` |
| Gateway port | `18789` (loopback) |
| Gateway mode | `local` |
| Gateway auth | `token` mode enabled |
| Systemd | Configured (`/etc/wsl.conf` has `systemd=true`) but **user bus not connecting** |
| Service file | `~/.config/systemd/user/openclaw-gateway.service` (exists, v2026.1.30) |
| Tailscale IP | `100.67.48.68` (active, hostname `asushadad-2`) |
| WSL2 NAT IP | `172.24.176.163/20` |
| RAM | 7.7 GiB allocated, ~5.3 GiB available |
| CPU cores | 8 |
| Agents | `main` (default), `career-hunter`, `monitor-active`, `hadad-bot` |
| Channels | Telegram (OK), WhatsApp (linked) |
| Skills | 15 eligible, 39 missing requirements |
| Plugins | 4 loaded, 27 disabled |

### Active Issues

| Issue | Details |
|-------|---------|
| Systemd bus failure | `Failed to connect to bus: No such file or directory` -- despite `systemd=true` in wsl.conf. May need `wsl --shutdown` and relaunch, or `loginctl enable-linger nehor`. |
| Service PATH mismatch | Gateway service PATH is missing `/home/nehor/.nvm/current/bin`. Run `openclaw doctor --fix` to regenerate. |
| Service version drift | Service file says v2026.1.30 but OpenClaw is v2026.2.3-1. Regenerate with `openclaw gateway install`. |
| Gateway auth warning | `doctor` reports gateway auth is off or missing a token (service token may differ from config token). |
| Punycode deprecation | Node v25.5.0 emits `DEP0040` warning for the `punycode` module. Cosmetic only; no action needed. |

### Recommended Fixes

```bash
# 1. Fix systemd (from PowerShell)
wsl --shutdown
# Then reopen Ubuntu and verify:
systemctl --user status

# 2. Regenerate gateway service with correct paths and version
openclaw gateway install

# 3. Enable and start the service
systemctl --user daemon-reload
systemctl --user enable --now openclaw-gateway.service

# 4. Verify
openclaw doctor
openclaw gateway status
```

---
*Source: https://docs.openclaw.ai/ + local system inspection -- Last updated 2026-02-08*
*Skill: ~/.claude/skills/openclaw-guide*
