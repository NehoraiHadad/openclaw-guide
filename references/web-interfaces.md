# Web Interfaces Reference

## Overview

OpenClaw provides several interfaces for interacting with and managing the agent system. All interfaces connect to the **Gateway**, the always-on process that owns channel connections and the control/event plane. The Gateway uses a single-port multiplex design (default port `18789`) serving both WebSocket and HTTP traffic.

| Interface | Type | Access | Primary Use |
|-----------|------|--------|-------------|
| Control UI (Dashboard) | Browser SPA | `http://127.0.0.1:18789/` | Full admin panel |
| TUI | Terminal | `openclaw tui` | Terminal-based chat & control |
| WebChat | Native app (macOS/iOS) | Menu bar app | Chat-focused sessions |
| HTTP API endpoints | REST | Same port as Gateway | Programmatic access |

---

## Control UI (Dashboard)

The Control UI is the primary browser-based admin interface. It is a single-page application built with **Vite + Lit** that communicates directly with the Gateway WebSocket on the same port.

### Accessing the Dashboard

**Local access:**

```
http://127.0.0.1:18789/
http://localhost:18789/
```

**Via CLI (recommended):**

```bash
openclaw dashboard
```

This command automatically copies the URL, opens the browser when possible, and displays SSH hints for headless environments.

**Custom base path:**

```json5
{
  gateway: {
    controlUi: {
      basePath: "/admin"   // Access at http://127.0.0.1:18789/admin/
    }
  }
}
```

### Building the Control UI

The Control UI is enabled by default when compiled assets are present in `dist/control-ui`. To build:

```bash
pnpm ui:build
```

### Features

The Control UI provides comprehensive administration capabilities:

| Feature Area | Capabilities |
|-------------|-------------|
| **Chat** | Messaging, tool call streaming, live output cards |
| **Channel management** | WhatsApp, Telegram, Discord, Slack status and QR login |
| **Instance monitoring** | Presence lists, refresh capability |
| **Session control** | Thinking/verbose overrides, session switching |
| **Cron jobs** | Creation, execution, history tracking |
| **Skill administration** | Status monitoring, installation, API key updates |
| **Configuration** | View/edit settings with validation and restart capability |
| **Debugging** | Status snapshots, event logs, RPC calls |
| **Live logging** | Gateway file logs with filtering and export |
| **System updates** | Package/git updates with restart reporting |

### Authentication

Authentication is enforced at the WebSocket handshake using `connect.params.auth`. Two methods are supported:

| Method | Configuration Key | Description |
|--------|------------------|-------------|
| Token | `gateway.auth.token` | Token string (default when env var is set) |
| Password | `gateway.auth.password` | Shared secret |

The UI stores credentials in browser `localStorage` after initial connection.

**Retrieving the token:**

```bash
openclaw config get gateway.auth.token
```

### Device Pairing

First-time connections from non-local addresses require **one-time pairing approval**. Local connections from `127.0.0.1` are auto-approved.

```bash
openclaw devices list                  # List pending/approved devices
openclaw devices approve <requestId>   # Approve a pairing request
```

### Troubleshooting Connection Issues

If you see "unauthorized" (WebSocket 1008 error):

1. Verify the gateway is reachable: `openclaw health`
2. Retrieve the current token: `openclaw config get gateway.auth.token`
3. Paste the token into the dashboard settings panel and reconnect

### Security Considerations

The Control UI is an **admin surface** and must never be exposed on the public internet without protection:

- Anti-clickjacking headers are applied automatically
- WebSocket connections restricted to same-origin unless `allowedOrigins` is configured
- Non-loopback binds require auth tokens/passwords
- Use Tailscale Serve or SSH tunnels for remote access (see Remote Access below)

---

## TUI (Terminal User Interface)

The TUI is a full-featured terminal interface for chatting with agents and managing sessions without a browser.

### Launching the TUI

```bash
# Start the gateway first
openclaw gateway

# Then open TUI (in another terminal)
openclaw tui

# Remote connection
openclaw tui --url ws://<host>:<port> --token <gateway-token>
```

### CLI Flags

| Flag | Description | Default |
|------|-------------|---------|
| `--url` | Gateway WebSocket URL | `ws://127.0.0.1:18789` |
| `--token` | Authentication token | (from config) |
| `--password` | Authentication password | (from config) |
| `--session` | Session key | `main` |
| `--deliver` | Enable provider delivery | off |
| `--thinking` | Override thinking level | (from config) |
| `--timeout-ms` | Agent timeout in ms | (from config) |
| `--history-limit` | Messages to load | `200` |

### UI Layout

The TUI displays four zones:

1. **Header:** Connection URL and active agent/session
2. **Chat log:** Messages and tool output cards
3. **Status line:** Connection state indicator
4. **Footer:** Metadata including token counts and delivery status

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Enter` | Send message |
| `Esc` | Abort active run |
| `Ctrl+C` | Clear input (press twice to exit) |
| `Ctrl+L` | Model picker |
| `Ctrl+G` | Agent picker |
| `Ctrl+P` | Session picker |
| `Ctrl+O` | Toggle tool output expansion |
| `Ctrl+T` | Toggle thinking visibility |

### Slash Commands

**Core commands:**

| Command | Purpose |
|---------|---------|
| `/help` | Show help |
| `/status` | Connection and session status |
| `/agent <id>` | Switch agent |
| `/session <key>` | Switch session |
| `/model <provider/model>` | Change model |

**Session controls:**

| Command | Purpose |
|---------|---------|
| `/think` | Toggle thinking mode |
| `/verbose` | Toggle verbose output |
| `/reasoning` | Toggle reasoning display |
| `/usage` | Show token usage |
| `/elevated` | Toggle elevated tools |
| `/activation` | Show activation info |
| `/deliver` | Toggle delivery mode |

**Lifecycle commands:**

| Command | Purpose |
|---------|---------|
| `/new` | Start new session |
| `/reset` | Reset current session |
| `/abort` | Abort current run |
| `/settings` | View/edit settings |
| `/exit` | Exit TUI |

### Agents & Sessions in the TUI

- **Agents** are unique slugs (e.g., `main`, `research`) exposed by the Gateway
- **Sessions** belong to the current agent, named as `agent:<agentId>:<sessionKey>`
- Sessions can operate in **per-sender** mode (multiple sessions per agent) or **global** scope (single shared session)

---

## WebChat (Native App)

WebChat is a native SwiftUI chat interface for macOS and iOS that connects directly to the Gateway WebSocket. It provides a lightweight, chat-focused experience without an embedded browser or local static server.

### Key Characteristics

- Native UI built with SwiftUI
- Uses the same sessions and routing rules as all other channels
- Deterministic routing: replies always return to WebChat
- Falls back to read-only mode if the gateway becomes unreachable

### Launching WebChat

**From the menu bar:**
Click the lobster menu icon and select "Open Chat"

**From the command line (for testing):**

```bash
dist/OpenClaw.app/Contents/MacOS/OpenClaw --webchat
```

### Connection Modes

| Mode | Description |
|------|-------------|
| **Local** | Direct connection to local Gateway WebSocket |
| **Remote** | SSH tunnel forwarding the Gateway control port |

### WebSocket Operations

WebChat communicates through three primary WebSocket methods:

| Method | Purpose |
|--------|---------|
| `chat.history` | Retrieve conversation history (always from gateway, never local) |
| `chat.send` | Send messages |
| `chat.inject` | Append assistant notes to transcript without triggering an agent run |

The app also monitors system events: chat updates, agent status, presence indicators, tick signals, and health metrics.

### Configuration

WebChat does not have a dedicated configuration block. It relies on existing gateway settings:

```json5
{
  gateway: {
    port: 18789,              // WebSocket port
    bind: "loopback",         // Bind address
    auth: {
      mode: "token",          // token|password
      token: "your-token"
    },
    remote: {
      url: "ws://...",        // Remote gateway URL
      token: "remote-token",
      password: "remote-pass"
    }
  }
}
```

### Session Management

- Defaults to the **main session** for the selected agent
- Includes a session switcher for navigating between sessions
- Uses a separate dedicated session for onboarding to isolate setup from chat

### Debugging

View WebChat logs:

```bash
./scripts/clawlog.sh
# Monitors subsystem: bot.molt, category: WebChatSwiftUI
```

---

## HTTP API Endpoints

The Gateway exposes REST-compatible API endpoints on the same multiplexed port:

| Endpoint | Protocol | Purpose |
|----------|----------|---------|
| `/v1/chat/completions` | OpenAI Chat Completions | Programmatic chat access |
| `/v1/responses` | OpenResponses | Response streaming |
| `/tools/invoke` | Tools Invoke | Direct tool execution |

---

## Gateway Port Architecture

The Gateway derives multiple service ports from a single base port (default `18789`):

| Port | Offset | Service |
|------|--------|---------|
| `18789` | Base (+0) | Gateway WebSocket + HTTP + Control UI |
| `18791` | Base +2 | Browser control service (loopback only) |
| `18793` | Base +4 | Canvas file server (workspace files) |
| `18798`-`18897` | Base +9 to +108 | CDP ports (auto-allocated per browser profile) |

Port precedence: CLI flag `--port` > environment variable > config file > default `18789`.

---

## Remote Access

### Option 1: Tailscale Serve (Recommended)

Tailscale Serve proxies the local gateway through your Tailnet with HTTPS. The gateway stays bound to loopback.

**Configuration:**

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: {
      mode: "serve"
    }
  }
}
```

**CLI:**

```bash
openclaw gateway --tailscale serve
```

**Access URL:** `https://<your-machine>.<tailnet-name>.ts.net/`

**Requirements:**
- HTTPS must be enabled for your tailnet (the CLI prompts if missing)

**Identity-based auth:** When `gateway.auth.allowTailscale: true`, valid Tailscale users authenticate via identity headers (`tailscale-user-login`) without supplying a token/password.

### Option 2: Tailscale Funnel (Public Access)

Funnel exposes the gateway to the public internet with HTTPS. A shared password is **mandatory**.

**Configuration:**

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: {
      mode: "funnel"
    },
    auth: {
      mode: "password",
      password: "strong-password-here"
    }
  }
}
```

**CLI:**

```bash
openclaw gateway --tailscale funnel --auth password
```

**Requirements:**
- Tailscale v1.38.3+
- MagicDNS enabled
- HTTPS enabled
- Funnel node attribute set
- Funnel only supports ports `443`, `8443`, and `10000` over TLS

### Option 3: Direct Tailnet Binding

Bind the gateway directly to the Tailscale network IP.

**Configuration:**

```json5
{
  gateway: {
    bind: "tailnet",
    auth: {
      mode: "token",
      token: "your-token"
    }
  }
}
```

### Option 4: SSH Tunnel

Forward the gateway port through SSH for remote access:

```bash
ssh -N -L 18789:127.0.0.1:18789 user@remote-host
```

**Auto-start with macOS Launch Agent:**

Create `~/Library/LaunchAgents/com.openclaw.tunnel.plist` to keep the tunnel alive:

- `KeepAlive: true` restarts the tunnel if it crashes
- `RunAtLoad: true` starts on login
- Runs in background without manual intervention

**Monitor tunnel status:**

```bash
ps aux | grep ssh
lsof -i :18789
```

**Restart/stop:**

```bash
launchctl kickstart gui/$(id -u)/com.openclaw.tunnel
launchctl bootout gui/$(id -u)/com.openclaw.tunnel
```

### Option 5: Remote Gateway Mode

Configure the CLI to connect to a remote gateway persistently:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",   // Via SSH tunnel
      token: "your-token",
      tlsFingerprint: "..."           // Pin TLS cert for wss://
    }
  }
}
```

When using `--url` on the CLI, explicitly pass `--token` or `--password`; implicit credential fallback is disabled.

### Tailscale Mode Summary

| Mode | Binding | Auth | Access Scope |
|------|---------|------|-------------|
| `serve` | Loopback | Tailscale identity or token | Tailnet only |
| `funnel` | Loopback | Password (required) | Public internet |
| `tailnet` | Tailscale IP | Token (required) | Tailnet only |
| Off (default) | Loopback | Token or none | Local only |

---

## Deployment Patterns

### 1. Loopback + Tailscale Serve (Recommended)

Gateway stays local; Tailscale handles remote proxying. Best balance of security and convenience.

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" }
  }
}
```

### 2. Persistent Gateway on VPS/Home Server

Run the Gateway on a dedicated host accessible via Tailscale or SSH. Recommended when your primary machine sleeps frequently. Keep `gateway.bind: "loopback"` and use Tailscale Serve for the Control UI.

### 3. Home Desktop Gateway

Your laptop connects remotely using the macOS app's "Remote over SSH" mode, which automatically manages SSH tunneling. The Gateway remains on your home machine.

### 4. Local Gateway with Remote Access

Keep the Gateway on your laptop but expose it through SSH tunnels or Tailscale Serve with loopback binding.

---

## Authentication Reference

### Auth Modes

| Mode | Config Key | When to Use |
|------|-----------|-------------|
| `token` | `gateway.auth.token` | Default; set via env var or config |
| `password` | `gateway.auth.password` | Shared secret for Funnel or multi-user |
| `none` | `gateway.auth.mode: "none"` | Local-only, single-user (not recommended) |

### Auth by Interface

| Interface | Auth Method | Notes |
|-----------|-------------|-------|
| Control UI | WebSocket handshake (`connect.params.auth`) | Stored in `localStorage` after first connect |
| TUI | `--token` or `--password` flag | Falls back to config |
| WebChat | Gateway config (`gateway.auth.*`) | Required even for local connections |
| HTTP API | Same as gateway auth | Token/password in request |

### Security Best Practices

- **Keep the gateway loopback-only** unless you are certain you need a bind
- Non-loopback binds always require auth tokens/passwords
- `gateway.remote.token` applies only to remote CLI calls, not local auth
- Use `gateway.remote.tlsFingerprint` to pin remote TLS certificates with `wss://`
- Tailscale Serve with `allowTailscale: true` provides identity-based auth without tokens

---

## Configuration Hot Reload

The Gateway watches `~/.openclaw/openclaw.json` (or `OPENCLAW_CONFIG_PATH`) for changes:

| Reload Mode | Behavior |
|-------------|----------|
| `hybrid` (default) | Applies safe changes immediately; restarts on critical updates |
| `off` | Disables automatic reload entirely |

Send `SIGUSR1` for an in-process restart when needed.

---

## Quick Reference: Interface Comparison

| Feature | Control UI | TUI | WebChat |
|---------|-----------|-----|---------|
| Platform | Any browser | Any terminal | macOS/iOS |
| Chat | Yes | Yes | Yes |
| Tool output | Streaming cards | Expandable cards | Streaming |
| Channel management | Yes | No | No |
| Configuration editing | Yes | Limited (`/settings`) | No |
| Cron management | Yes | No | No |
| Skill management | Yes | No | No |
| Live logs | Yes | No | No |
| Debug tools | Yes | No | No |
| System updates | Yes | No | No |
| Session switching | Yes | Yes (`Ctrl+P`) | Yes |
| Agent switching | Yes | Yes (`Ctrl+G`) | Yes |
| Model switching | Yes | Yes (`Ctrl+L`) | No |
| Offline fallback | No | No | Read-only mode |
| Keyboard shortcuts | Browser defaults | Full set | macOS native |

---
*Source: https://docs.openclaw.ai/ — Last updated 2026-02-08*
*Skill: ~/.claude/skills/openclaw-guide*
