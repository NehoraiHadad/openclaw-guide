# Skills System

## Overview

OpenClaw uses AgentSkills-compatible skill folders to extend agent capabilities. Each skill is a directory containing a `SKILL.md` file with YAML frontmatter and instructions.

## SKILL.md Format

### Required Fields

```yaml
---
name: skill-identifier
description: Brief functionality description
---

# Skill content here
```

**Important:** The parser supports **single-line** frontmatter keys only. Metadata must be a single-line JSON object.

### Optional Frontmatter

| Parameter | Values | Purpose |
|-----------|--------|---------|
| `homepage` | URL | Shown as "Website" in macOS Skills UI |
| `user-invocable` | true/false (default: true) | Exposes skill as slash command |
| `disable-model-invocation` | true/false (default: false) | Excludes from model prompt |
| `command-dispatch` | tool | Bypasses model, dispatches directly |
| `command-tool` | tool name | Specifies tool for dispatch |
| `command-arg-mode` | raw (default) | Forwards raw args to tool |

### Metadata for Gating

```yaml
metadata: {"openclaw":{"requires":{"bins":["jq"],"env":["API_KEY"],"config":["browser.enabled"]},"os":["darwin"]}}
```

## Gating & Load-Time Filtering

Skills filter via `metadata.openclaw` JSON object:

### Availability Controls

| Field | Description |
|-------|-------------|
| `always: true` | Include skill regardless of other gates |
| `os` | List of eligible platforms: `darwin`, `linux`, `win32` |

### Dependency Requirements

| Field | Description |
|-------|-------------|
| `requires.bins` | All listed binaries must exist on PATH |
| `requires.anyBins` | At least one binary must exist |
| `requires.env` | Environment variables (checked in config or process) |
| `requires.config` | Paths in openclaw.json that must be truthy |

### Additional Metadata

| Field | Description |
|-------|-------------|
| `emoji` | Used by macOS Skills UI |
| `primaryEnv` | Associates env variable with `skills.entries.<name>.apiKey` |

## Skill Loading Hierarchy

Three locations with hierarchical precedence (highest to lowest):

1. **Workspace skills** (`<workspace>/skills`)
2. **Managed/local skills** (`~/.openclaw/skills`)
3. **Bundled skills** (shipped with install)

**Additional layers:**
- Plugin skills participate in normal precedence
- Extra directories via `skills.load.extraDirs` have lowest precedence

**Multi-agent context:**
- Per-agent skills in workspace
- Shared skills in managed/local directory
- Name conflicts resolved by precedence hierarchy

## Installation & Installers

Skills can include installer specs for automated setup.

### Installer Kinds

| Kind | Description |
|------|-------------|
| `brew` | Homebrew formula |
| `node` | npm/pnpm/yarn/bun package |
| `go` | Go module |
| `uv` | Python uv package |
| `download` | Direct file download |

### Installer Metadata

```yaml
metadata: {"openclaw":{"installers":[{"id":"brew-jq","kind":"brew","bins":["jq"],"label":"jq via Homebrew"}]}}
```

**Download installer options:** `url`, `archive` (tar.gz/tar.bz2/zip), `stripComponents`, `targetDir`

## Configuration

### Per-Skill Settings

```json5
{
  skills: {
    entries: {
      "skill-name": {
        enabled: true,        // false disables permanently
        apiKey: "SECRET",     // maps to primaryEnv
        env: { VAR: "value" }, // inject if not already set
        config: { key: "value" }
      }
    }
  }
}
```

### Loading Settings

```json5
{
  skills: {
    load: {
      watch: true,           // hot reload on SKILL.md changes
      watchDebounceMs: 250,
      extraDirs: ["/path/to/skills"]
    },
    install: {
      nodeManager: "npm"     // npm/pnpm/yarn/bun
    },
    allowBundled: ["skill1", "skill2"]  // optional allowlist
  }
}
```

## Environment Injection

Per agent run:
1. Read skill metadata requirements
2. Apply `skills.entries.<key>.env` or `apiKey` to process environment
3. Build system prompt with eligible skills
4. Restore original environment after run

**Scope:** Agent run only (not global shell environment)

**Security:** Secrets injected into host process for that turn; keep secrets from prompts/logs.

## Performance & Runtime

### Session Snapshots

- Eligible skills captured when session starts
- Reused across subsequent turns
- Changes take effect on new session

### Skills Watcher

- Monitors skill folders for SKILL.md changes
- Auto-refreshes eligible skills list (hot reload)
- Configured via `skills.load.watch`

### Token Cost

Skills add deterministic overhead to prompts:

**Formula:** 195 base chars + Σ(97 + escaped_name + escaped_description + escaped_location)

**Estimates:**
- Base overhead: ~49 tokens
- Per skill: ~24 tokens plus field lengths
- XML escaping expands `& < > " '`

## ClawdHub Registry

Public skills marketplace: https://clawdhub.com

### Commands

```bash
clawdhub install <skill-slug>   # Install to ./skills
clawdhub update --all           # Update all installed
clawdhub sync --all             # Scan and publish updates
clawdhub search "keyword"       # Search registry
```

Default: installs to current directory's `./skills`

## Plugin Skills

Plugins can ship skills by listing directories in `openclaw.plugin.json`:

```json5
{
  skills: ["skills/my-skill", "skills/another-skill"]
}
```

Plugin skills participate in normal precedence and can be gated via config requirements.

## CLI Commands

```bash
openclaw skills                     # List all skills
openclaw skills show <skill-name>   # Show details
openclaw skills enable <skill-name>
openclaw skills disable <skill-name>
```

## Security Considerations

- Treat third-party skills as **trusted code** — review before enabling
- Prefer sandboxed runs for untrusted inputs
- Binary requirements checked on host at load time
- Sandboxed agents need binaries in container via `setupCommand`
- Secrets from `skills.entries.*.env` and `*.apiKey` injected into host process

## Creating Skills

### Directory Structure

```
skill-name/
├── SKILL.md           # Required
├── scripts/           # Optional executables
├── references/        # Optional documentation
└── assets/            # Optional files for output
```

### Minimal Example

```yaml
---
name: my-skill
description: Does something useful
---

# My Skill

Instructions for using this skill.
```

### With Gating

```yaml
---
name: browser-skill
description: Browser automation helpers
metadata: {"openclaw":{"requires":{"bins":["chromium"],"config":["browser.enabled"]}}}
---

# Browser Skill

Requires browser enabled and Chromium installed.
```

### Use `{baseDir}` for Paths

Reference skill folder paths with `{baseDir}` placeholder.

---

## Bot-Orchestrated Workflow Skills

A powerful pattern: skills that teach the bot to **orchestrate workflows** using its existing tools, rather than creating new scripts.

### The Pattern

Instead of writing new automation scripts, create a skill that instructs the bot to:
1. Run an existing script via `exec` tool
2. Parse the output (JSON, text, etc.)
3. Use native bot tools (`message`, `poll`, etc.) to act on results

### Example: WhatsApp Group Survey

```yaml
---
name: whatsapp-survey
description: Send WhatsApp group surveys to Telegram for curation
user-invocable: true
---

# WhatsApp Group Survey Skill

When the user asks to see WhatsApp groups:

## Step 1: Run the filter script
Use exec tool to run:
\`\`\`bash
node ~/clawd/scripts/whatsapp/filter-groups.mjs personal --cached --json [--filter "keyword"]
\`\`\`

## Step 2: Parse the output
The script outputs a JSON array of groups.

## Step 3: Send polls to the user
For each batch of up to 10 groups, use the message tool to send a poll.

## Step 4: Save the mapping
Write the number→group mapping for later processing.
```

### Benefits of This Approach

1. **No new scripts needed** - Reuses existing tools and scripts
2. **Bot handles messaging** - Uses native `message` tool (more reliable than custom scripts)
3. **Flexible** - Bot can adapt to different inputs, filters, edge cases
4. **Maintainable** - Just one SKILL.md file to update

### Best Practices for Workflow Skills

| Practice | Why |
|----------|-----|
| **No hardcoded user IDs** | Skills may be shared publicly |
| **Use general terms** | "send to the user" not "send to 12345" |
| **Reference existing scripts** | Don't duplicate logic |
| **Let bot adapt** | Write instructions, not rigid procedures |
| **Include example flows** | Show the expected interaction pattern |

### When to Use This Pattern

- Workflows that combine script output with bot messaging
- Tasks that need human-in-the-loop (polls, confirmations)
- Operations that benefit from bot's ability to adapt to edge cases
- Multi-step processes that span tools (exec → parse → message)

---

## Continuous Improvement

**Always verify with current docs:** Before implementing, fetch the relevant page from https://docs.molt.bot/tools/skills to check for updates.

When using OpenClaw and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/openclaw-guide/references/skills-system.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage
