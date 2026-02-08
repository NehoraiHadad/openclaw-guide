# Browser Automation Tools

## Overview

OpenClaw provides a separate, agent-only browser through a managed Chrome/Brave/Edge/Chromium profile isolated from personal browsing.

## Browser Profiles

Three profile types supported:

### Clawd-Managed (Isolated)

- Dedicated Chromium instance with separate user data directory
- Orange accent by default
- No extension required
- CDP ports auto-assigned from 18800–18899

### Chrome Extension Relay

- Drives existing Chrome tabs via local relay + extension
- Requires manual tab attachment
- Relay listens at default `http://127.0.0.1:18792`
- Better for preserving personal browsing context

### Remote CDP

- External Chromium-based browser via explicit CDP URL
- Supports HTTPS endpoints with authentication tokens
- Useful for browserless services or distributed setups

## Snapshot Modes

### AI Snapshot (Default)

Format: `openclaw browser snapshot`

- Produces text representation with numeric identifiers like `[ref=12]`
- Actions reference these numbers: `click 12`, `type 23 "text"`
- Requires Playwright installation

### Role Snapshot

Format: `snapshot --interactive` or with `--compact`, `--depth`, `--selector`, `--frame`

- Returns accessibility tree with role-based references (`e12`)
- Actions use format: `click e12`, `highlight e12`
- Useful for inspection-only without Playwright

### Snapshot Options

| Flag | Description |
|------|-------------|
| `--efficient` | Compact preset (interactive + compact + depth + lower char limits) |
| `--labels` | Viewport screenshot with overlaid ref labels |
| `--frame "<selector>"` | Scope to specific iframe |
| `--format aria` | Raw accessibility tree (no refs) |
| `--limit N` | Constrain output size |

**Important:** Refs are unstable across page navigations; regenerate snapshot after navigation.

## Action Commands

### Navigation & Tabs

| Command | Description |
|---------|-------------|
| `navigate <url>` | Direct page load |
| `tab new` | Create fresh tab |
| `tab select <number>` | Focus specific tab |
| `tab close <id>` | Close by target ID |
| `open <url>` | Open URL in current tab |
| `focus <targetId>` | Bring tab into focus |
| `close <targetId>` | Close specific tab |

### User Input Actions

| Command | Description |
|---------|-------------|
| `click <ref>` / `--double` | Click element (numeric or role ref) |
| `type <ref> "text"` / `--submit` | Type and optionally submit |
| `press <key>` | Send keyboard input (e.g., Enter) |
| `hover <ref>` | Move cursor to element |
| `scrollintoview <ref>` | Scroll element into viewport |
| `drag <ref1> <ref2>` | Drag-and-drop between elements |
| `select <ref> <option1> <option2>` | Multi-select dropdown |
| `fill --fields '[{"ref":"1","type":"text","value":"text"}]'` | Batch fill |

### File Operations

| Command | Description |
|---------|-------------|
| `upload <path>` | Arm file chooser (call before triggering action) |
| `download <ref> <path>` | Save file from element |
| `waitfordownload <path>` | Block until download completes |

### Inspection & Debugging

| Command | Description |
|---------|-------------|
| `screenshot` / `--full-page` | Capture pixels |
| `screenshot --ref <ref>` | Element-specific screenshot |
| `console --level <level>` | View console logs |
| `errors` / `--clear` | Browser errors |
| `requests --filter <type>` / `--clear` | Network requests |
| `pdf` | Generate PDF of page |
| `responsebody "<pattern>" --max-chars <n>` | Inspect response bodies |
| `trace start` / `trace stop` | Record DevTools trace |
| `highlight <ref>` | Visually mark element |

### Advanced Waiting

| Command | Description |
|---------|-------------|
| `wait --text "string"` | Wait for text appearance |
| `wait "#selector"` | Wait for element visibility |
| `wait --url "**/path"` | Wait for URL match (glob) |
| `wait --load networkidle` | Wait for network idle state |
| `wait --fn "window.condition===true"` | Wait for JS predicate |
| `wait --timeout-ms <ms>` | Custom timeout |

### State Management

| Command | Description |
|---------|-------------|
| `resize <width> <height>` | Set viewport |
| `evaluate --fn '<js>' --ref <ref>` | Execute JavaScript |
| `dialog --accept` / `--dismiss` | Handle alert/confirm dialogs |

## State Management Commands

### Cookies

```bash
cookies                              # List all
cookies set <name> <value> --url "https://..."  # Create
cookies clear                        # Remove all
```

### Storage

```bash
storage local|session get            # Retrieve
storage local|session set <key> <value>  # Set
storage local|session clear          # Clear all
```

### Network & Device Simulation

| Command | Description |
|---------|-------------|
| `set offline on\|off` | Simulate offline mode |
| `set headers --json '{"Header":"Value"}'` | Custom headers |
| `set credentials <user> <pass>` | HTTP Basic auth |
| `set geo <lat> <lon> --origin "https://..."` | Geolocation |
| `set media dark\|light\|no-preference\|none` | Color scheme |
| `set timezone <tz>` | Override timezone |
| `set locale <locale>` | Override language |
| `set device "<name>"` | Apply Playwright device preset |
| `set viewport <width> <height>` | Resize viewport |

## Configuration

```json5
{
  browser: {
    enabled: true,
    remoteCdpTimeoutMs: 1500,
    remoteCdpHandshakeTimeoutMs: 3000,
    defaultProfile: "clawd",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/path/to/browser",
    evaluateEnabled: false,  // Disable arbitrary JS for security
    profiles: {
      clawd: {
        cdpPort: 18800,
        color: "#FF4500"
      },
      work: {
        cdpPort: 18801,
        color: "#0066CC"
      },
      remote: {
        cdpUrl: "http://10.0.0.42:9222",
        color: "#00AA00"
      },
      browserless: {
        cdpUrl: "https://production-sfo.browserless.io?token=<KEY>",
        color: "#00AA00"
      }
    },
    snapshotDefaults: {
      mode: "efficient"
    }
  }
}
```

### Port Allocation

- Control service: `gateway.port + 9` (default 18791)
- Relay: `gateway.port + 10` (default 18792)
- Local CDP: 18800–18899

### Browser Auto-Detection Priority

1. System default (if Chromium-based)
2. Chrome
3. Brave
4. Edge
5. Chromium
6. Chrome Canary

## Chrome Extension Setup

```bash
openclaw browser extension install
```

1. Navigate to `chrome://extensions`
2. Enable Developer mode
3. Load unpacked from printed directory
4. Pin extension
5. Click extension icon to attach tab (badge shows `ON`)
6. Use `profile="chrome"` in browser tool

## Remote & Browserless Integration

### Browserless Service

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    remoteCdpTimeoutMs: 2000,
    remoteCdpHandshakeTimeoutMs: 4000,
    profiles: {
      browserless: {
        cdpUrl: "https://production-sfo.browserless.io?token=<KEY>",
        color: "#00AA00"
      }
    }
  }
}
```

### Remote CDP Authentication

- Query tokens: `https://provider.example?token=<token>`
- Basic auth: `https://user:pass@provider.example`
- Prefer environment variables over config files

## Security & Privacy

### Isolation Guarantees

- Separate user data directory (never touches personal profile)
- Dedicated CDP ports (avoids dev tool conflicts)
- Deterministic tab control by ID

### Key Protections

- Loopback-only control; access flows through Gateway auth or node pairing
- Keep Gateway and nodes on private network
- Treat remote CDP URLs/tokens as secrets
- Prefer HTTPS endpoints and short-lived tokens

### Risks

- `evaluate` / `wait --fn` execute arbitrary JS; prompt injection risk
- Disable with `browser.evaluateEnabled=false` if not needed
- Remote CDP is powerful; tunnel and protect access

## CLI Quick Reference

```bash
openclaw browser status
openclaw browser start / stop
openclaw browser tabs
openclaw browser tab new / select <n> / close <id>
openclaw browser open <url> / navigate <url>
openclaw browser snapshot / screenshot
openclaw browser click <ref> / type <ref> "text"
openclaw browser profiles
openclaw browser create-profile --name <name> --color "<hex>"
```

All commands accept `--browser-profile <name>` and `--json`.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Browser disabled" | Enable in config and restart Gateway |
| "Playwright not available" | Install full `playwright` package |

**Debug workflow:**
1. `openclaw browser snapshot --interactive`
2. Use role refs (`e12`) in click/type
3. If fails: `openclaw browser highlight <ref>`
4. Check errors: `--clear` to reset logs
5. Record trace: `trace start` → reproduce → `trace stop`

## Sandboxed Sessions

When session is sandboxed, browser defaults to `target="sandbox"`. For Chrome extension:
- Run session unsandboxed, or
- Set `agents.defaults.sandbox.browser.allowHostControl: true` and use `target="host"`

---

## Continuous Improvement

**Always verify with current docs:** Before implementing, fetch the relevant page from https://docs.molt.bot/tools/browser to check for updates.

When using OpenClaw and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/openclaw-guide/references/tools-browser.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage
