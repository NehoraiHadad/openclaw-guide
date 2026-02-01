---
name: OpenClaw Guide
description: "PROACTIVE LEARNING SKILL - Use this skill AND update it during work. Trigger when user mentions: openclaw, clawdbot, moltbot, molt.bot, lobster bot, gateway, openclaw.json, clawdbot.json, moltbot.json, channel config, WhatsApp bot, Telegram bot, Discord bot, Slack bot, iMessage, Signal, exec tool, browser tool, sessions, sub-agents, multi-agent, cron, webhooks, agent routing, sandbox, skills, ClawdHub, heartbeat, bootstrap files, AGENTS.md, SOUL.md, memory files. Also trigger on: gateway restart, doctor command, channel login, session management, context overflow, cooldown, auth profiles, model fallbacks. This is a LEARNING SKILL: when you discover new OpenClaw behavior or fix issues, IMMEDIATELY update this skill file."
version: 0.3.2
---

# OpenClaw Complete Guide (formerly Clawdbot/Moltbot)

---

## 🧠 THIS IS A LEARNING SKILL — UPDATE IT PROACTIVELY!

**This skill learns and improves through use. You MUST update it during work, not just when asked.**

### CRITICAL: Update During Work, Not After

When working on any OpenClaw task:
1. **Before finishing** - Check if you learned something new
2. **After fixing a bug** - Document the fix immediately
3. **After reading docs** - Update if docs differ from this skill
4. **After user corrects you** - Fix this skill right away

**Don't wait for the user to ask. Update proactively.**

### When to Update This Skill

**IMMEDIATELY update when you:**
- Discover something new about OpenClaw (feature, behavior, config option)
- Find that documented info here is wrong or outdated
- Learn a workaround or best practice from real usage
- Get corrected by docs or by the user
- Notice the official docs differ from what's written here
- Fix a problem (document the solution!)
- See an error message and its resolution

### How to Update

1. **Reference files** (`~/.claude/skills/openclaw-guide/references/`): Add detailed technical info
2. **This SKILL.md**: Update overview sections or add new categories
3. **Be specific**: Include examples, config snippets, exact behavior observed
4. **Note the source**: "Learned from user on 2026-02-01" or "Per docs.openclaw.ai/..."

### Why This Matters

Without updates, this skill becomes stale. The platform evolves constantly. If you use OpenClaw knowledge and don't write it back here, future sessions lose that knowledge.

**Text > Brain. Write it down. Now, not later.**

---

## IMPORTANT: Always Verify with Current Documentation

**OpenClaw is a rapidly evolving platform.** This skill may contain outdated information.

**Before implementing any solution:**
1. **Fetch the current documentation** for the specific topic from https://docs.openclaw.ai/
2. Use WebFetch on the relevant docs page to get the latest information
3. The LLM-friendly reference at https://docs.openclaw.ai/llms.txt contains the full documentation index
4. **Use local docs:** `openclaw docs <topic>` for offline access

**Common documentation URLs:**
- Channels: `https://docs.openclaw.ai/channels/<channel-name>` (e.g., `/channels/whatsapp`)
- Tools: `https://docs.openclaw.ai/tools/<tool-name>` (e.g., `/tools/exec`, `/tools/browser`)
- CLI: `https://docs.openclaw.ai/cli/<command>` (e.g., `/cli/sessions`, `/cli/cron`)
- Concepts: `https://docs.openclaw.ai/concepts/<topic>` (e.g., `/concepts/multi-agent`)

---

## Overview

**OpenClaw** (formerly Clawdbot/Moltbot) is a self-hosted AI assistant platform that operates through a local gateway architecture. It enables interaction with Claude or other AI models through multiple communication channels.

**Key characteristics:**
- Local-first gateway serving as unified control plane
- WebSocket-based architecture coordinating sessions, channels, tools, and events
- Multi-channel support (WhatsApp, Telegram, Discord, Slack, iMessage, Signal, etc.)
- Isolated agent workspaces with flexible routing
- Skills system for extensibility
- Voice integration with Wake and Talk modes

**Installation:**
```bash
npm install -g openclaw@latest
openclaw onboard
```

Requires Node.js 22+ (Node.js 24 recommended). Installs as daemon service (launchd on macOS, systemd on Linux).

**Current Version:** 2026.1.30 (as of 2026-02-01)

## Quick Reference

### Configuration Location

Primary config: `~/.openclaw/openclaw.json`

**Legacy paths (still supported via symlinks after migration):**
- `~/.clawdbot/clawdbot.json` → symlinked to `~/.openclaw/openclaw.json`
- State dir: `~/.clawdbot` → symlinked to `~/.openclaw`

### Common Commands

| Command | Purpose |
|---------|---------|
| `openclaw doctor` | Run diagnostics and repairs |
| `openclaw sessions` | List conversation sessions |
| `openclaw channels login` | Set up channel authentication |
| `openclaw cron` | Manage scheduled jobs |
| `openclaw browser` | Control browser automation |
| `openclaw gateway restart` | Restart the gateway |
| `openclaw configure` | Re-run configuration/auth setup |

**Note:** Use `openclaw` command (formerly `clawdbot`). Top-level `restart` doesn't exist - use `gateway restart`.

### Dashboard Access

Local dashboard: `http://127.0.0.1:18789/`

## Tool Categories

### Core Execution Tools

The `exec` tool runs shell commands with configurable security.

**Key parameters:**
- `command` - Shell command to execute
- `host` - Execution location: `sandbox` (default), `gateway`, or `node`
- `security` - Enforcement: `deny`, `allowlist`, `full`
- `elevated` - Request elevated permissions

**Reference:** See `references/tools-core.md` for complete exec tool documentation.

### Browser Automation

Agent-controlled browser with two modes:
- **Clawd (Managed):** Dedicated browser instance with CDP control
- **Chrome (Extension Relay):** Controls existing Chrome tabs via extension

**Reference:** See `references/tools-browser.md` for browser automation details.

### Session & Sub-agents

- **Sessions:** Isolated conversation contexts per channel/group
- **Sub-agents:** Background agent runs for parallel tasks

**Reference:** See `references/tools-sessions.md` for session management.

### Automation

- **Cron:** Scheduled jobs via gateway scheduler
- **Webhooks:** HTTP endpoints for external triggers
- **Polling:** Periodic checks for external events

**Reference:** See `references/tools-automation.md` for automation setup.

## Supported Channels

### Messaging Platforms

| Channel | Auth Method | Key Config |
|---------|-------------|------------|
| WhatsApp | QR code via Baileys | `channels.whatsapp` |
| Telegram | BotFather token | `channels.telegram.botToken` |
| Discord | Bot token | `channels.discord.token` |
| Slack | App + Bot tokens | `channels.slack` |
| Signal | signal-cli | `channels.signal` |

**Reference:** See `references/channels-messaging.md` for detailed setup.

### Apple Platforms

iMessage integration via `imsg` CLI tool (macOS only).

**Reference:** See `references/channels-apple.md` for iMessage setup.

## Agent Architecture

### Workspace Structure

Each agent maintains:
- Dedicated workspace directory
- State directory (`agentDir`)
- Session store
- Bootstrap files (AGENTS.md, SOUL.md, USER.md, etc.)

### Bootstrap Files

| File | Purpose | Loaded When |
|------|---------|-------------|
| AGENTS.md | Operating instructions | Every session |
| SOUL.md | Persona, boundaries, tone | Every session |
| USER.md | User profile | Every session |
| TOOLS.md | Local tool notes | Every session |
| IDENTITY.md | Agent name, emoji | Every session |
| HEARTBEAT.md | Heartbeat checklist | On heartbeat |
| BOOT.md | Gateway restart checklist | On boot |
| BOOTSTRAP.md | First-run ritual | First run only (then deleted) |
| MEMORY.md | Long-term memory | Main session only |
| memory/YYYY-MM-DD.md | Daily logs | Today + yesterday |

### Default Templates

OpenClaw ships with default templates at:
```
~/.nvm/.../openclaw/docs/reference/templates/
```

Use `openclaw setup` to seed missing files, or copy templates manually.

**Config options:**
- `agents.defaults.skipBootstrap: true` — Don't inject bootstrap files
- `agents.defaults.bootstrapMaxChars: 20000` — Truncation limit

**Reference:** See `references/agent-architecture.md` and `references/agent-behavior.md` for details.

### Multi-Agent Routing

Routing uses deterministic, most-specific-wins matching:
1. Direct peer matches (DM/group/channel ID)
2. Guild ID (Discord)
3. Team ID (Slack)
4. Account ID matching
5. Channel-wide fallback
6. Default agent assignment

**Reference:** See `references/agent-architecture.md` for routing rules.

## Configuration

### openclaw.json Structure

```json5
{
  agents: {
    defaults: {
      workspace: "~/workspace",
      model: "google-gemini-cli/gemini-3-pro-preview",
      timeoutSeconds: 600,  // Official default, increase for large contexts
      sandbox: { mode: "off" }
    },
    list: [{ id: "main", /* agent config */ }]
  },
  channels: {
    whatsapp: { enabled: true, /* ... */ },
    telegram: { enabled: true, botToken: "..." }
  },
  tools: {
    exec: { pathPrepend: "/custom/bin" },
    elevated: { enabled: true, allowFrom: [...] }
  },
  skills: {
    entries: { "skill-name": { enabled: true } }
  }
}
```

**Important defaults:**
- `timeoutSeconds`: 600 (10 minutes) - official default, increase for 100k+ token contexts

**Reference:** See `references/configuration.md` for complete config reference.

## Model Configuration (Learned 2026-02-01)

### Available Gemini Models

| Model | Speed | Quality | Notes |
|-------|-------|---------|-------|
| `google-gemini-cli/gemini-2.5-flash` | Fast | Good | Recommended for responsive bots |
| `google-gemini-cli/gemini-2.5-pro` | Slow | Excellent | Can hang on some machines |
| `google-gemini-cli/gemini-3-flash-preview` | Fast | Good | Experimental, newest |
| `google-gemini-cli/gemini-3-pro-preview` | Slow | Excellent | Experimental, newest |

### Changing Models

**Method 1: CLI (fastest)**
```bash
openclaw config set agents.defaults.model.primary google-gemini-cli/gemini-3-flash-preview
openclaw gateway restart
```

**Method 2: Edit config directly**
```bash
nano ~/.openclaw/openclaw.json
# Edit agents.defaults.model.primary
openclaw gateway restart
```

**Method 3: Dashboard**
Open `http://127.0.0.1:18789/` in browser

### Model Fallback Configuration

```json5
{
  agents: {
    defaults: {
      model: {
        "primary": "google-gemini-cli/gemini-3-flash-preview",
        "fallbacks": [
          "google-gemini-cli/gemini-3-pro-preview",
          "google-gemini-cli/gemini-2.5-flash",
          "google-gemini-cli/gemini-2.5-pro"
        ]
      }
    }
  }
}
```

**IMPORTANT:** Fallback only triggers on **failures/timeouts**, NOT on slowness. If the primary model is slow but responding, you'll wait the full `timeoutSeconds` (default: 600s = 10 minutes) before fallback kicks in.

### Recommendation

For responsive bots, use Flash models as primary:
- `gemini-3-flash-preview` (newest, experimental)
- `gemini-2.5-flash` (stable, reliable)

Use Pro models only when quality is critical and you can tolerate delays.

## Skills System

Skills extend agent capabilities through SKILL.md files.

**Loading hierarchy (highest to lowest):**
1. Workspace skills: `<workspace>/skills`
2. Managed skills: `~/.openclaw/skills` (legacy: `~/.clawdbot/skills`)
3. Bundled skills: shipped with installation

**ClawdHub:** Public skills marketplace at https://clawdhub.com

```bash
clawdhub install <skill-slug>
clawdhub update --all
```

**Reference:** See `references/skills-system.md` for skill development.

## Memory System

### Daily Logs

Stored in `memory/YYYY-MM-DD.md` - append-only notes loaded at session start.

### Long-term Memory

`MEMORY.md` contains curated durable facts, loaded only in private sessions.

### Vector Search

Optional semantic search across markdown files using embeddings.

**Reference:** See `references/memory-sessions.md` for memory details.

### Memory Behavior

Memory behavior (when/how to write files) is defined in the workspace's `AGENTS.md` file, not hardcoded by the platform.

**Platform provides:**
- File loading mechanism (which files, when)
- Automatic memory flush before compaction
- Vector search over memory files

**Workspace defines:**
- What to write and when (user's AGENTS.md)
- Specific behaviors and rules (user's templates)

**Reference:** See `references/agent-behavior.md` for platform mechanics, `references/memory-sessions.md` for configuration.

## Heartbeat System

Heartbeats are periodic polls configured via `agents.defaults.heartbeat`.

**Config:**
```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "3600",
        model: "anthropic/claude-sonnet-4",
        target: "whatsapp",
        to: "+15551234567"
      }
    }
  }
}
```

**What happens:**
1. Gateway sends heartbeat prompt to agent
2. Agent reads `HEARTBEAT.md` (if exists)
3. Agent acts on checklist or replies `HEARTBEAT_OK`

**Heartbeat vs Cron:** Use heartbeat for batched checks with context; use cron for precise timing and isolated tasks.

**Reference:** See `references/agent-behavior.md` for mechanics, `references/tools-automation.md` for cron.

## Group Chat Configuration

### Mention Detection

**Agent-level patterns:**
```json5
{
  agents: {
    list: [{
      id: "main",
      groupChat: { mentionPatterns: ["@clawd", "hey clawd"] }
    }]
  }
}
```

**Channel-level requireMention:**
```json5
{
  channels: {
    whatsapp: {
      accounts: {
        personal: {
          groups: {
            "*": { requireMention: true },
            "120363...@g.us": { requireMention: false }
          }
        }
      }
    }
  }
}
```

### Platform Formatting Constraints

| Platform | Constraint |
|----------|------------|
| Discord | 2000 char max, embeds on links |
| WhatsApp | No markdown headers |
| Discord/WhatsApp | Tables don't render |

**Reference:** See `references/agent-architecture.md` for routing, `references/agent-behavior.md` for platform constraints.

## Common Patterns

### Basic WhatsApp Setup

```json5
{
  channels: {
    whatsapp: {
      enabled: true,
      dmPolicy: "pairing",
      selfChatMode: false
    }
  }
}
```

Then run: `openclaw channels login`

### Enable Sandboxing for Groups

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "non-main" }
    }
  }
}
```

### Configure Elevated Mode

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: ["+15551234567"]
    }
  }
}
```

## Custom Scripts Location

Custom scripts for operations not supported by Moltbot's API can be placed in the workspace scripts folder:

```
<workspace>/scripts/
└── whatsapp/
    ├── list-groups.mjs      # List all WhatsApp groups
    ├── filter-groups.mjs    # Filter groups by keywords
    └── passive-monitor.mjs  # Monitor messages passively
```

**Default location:** `<workspace>/scripts/whatsapp/` (wherever `agents.defaults.workspace` points)

These scripts use direct Baileys access for operations like:
- Listing all WhatsApp groups (not available via Moltbot API)
- Filtering groups by name/size
- Passive message monitoring without agent response

See `references/baileys-direct-access.md` for script documentation and available Baileys functions.

## Known Limitations & Workarounds

### WhatsApp Passive Monitoring

**Problem:** No built-in way to receive/log WhatsApp messages without the agent responding.

| Policy Setting | Messages Logged? | Agent Responds? |
|----------------|------------------|-----------------|
| `groupPolicy: "open"` | Yes | Yes (on mention) |
| `groupPolicy: "allowlist"` (empty) | No | No |
| `groupPolicy: "disabled"` | No | No |

**Workaround:** Use direct Baileys access for passive monitoring. See `references/baileys-direct-access.md`.

### WhatsApp Group Discovery

**Problem:** `openclaw directory groups list` only reads from config, not live WhatsApp data.

**Workaround:** Use the Baileys scripts in the workspace:
```bash
# List all groups (briefly disconnects gateway)
node <workspace>/scripts/whatsapp/list-groups.mjs <account>

# Filter by keywords (uses cached data, no reconnect)
node <workspace>/scripts/whatsapp/filter-groups.mjs <account> --cached --filter keyword1,keyword2
node <workspace>/scripts/whatsapp/filter-groups.mjs <account> --cached --min-participants 50
```

See `references/baileys-direct-access.md` for full documentation.

### Telegram vs WhatsApp Action Gating

Telegram supports `actions.sendMessage: false` to prevent sending messages while still receiving. WhatsApp does NOT have this feature.

### Gemini API Hanging Without Timeout (Learned 2026-02-01)

**Problem:** Bot shows "typing..." indicator but never responds. Logs show request started but no completion or error.

**Cause:** Gemini API (especially Pro models) can hang indefinitely without returning a response or triggering a timeout. The typing indicator stops after 2 minutes (TTL), but the request continues waiting.

**Symptoms:**
- Typing indicator appears for 2 minutes then disappears
- No response from bot
- Logs show `embedded run start` but no `embedded run complete`
- Request hangs for 10+ minutes

**Solutions:**
1. **Use Flash models** - More responsive than Pro models:
   - `gemini-3-flash-preview` or `gemini-2.5-flash` as primary
2. **Reduce timeout** - Set lower `timeoutSeconds` to trigger fallback sooner:
   ```json
   "timeoutSeconds": 120
   ```
3. **Restart gateway** - Cancels stuck requests:
   ```bash
   openclaw gateway restart
   ```

**Note:** Fallback only triggers on timeout/error, not slowness. If the API is slow but not failing, you wait the full timeout.

### Context Overflow Despite Config Change (Learned 2026-01-31)

**Problem:** After increasing `contextTokens` in config, sessions still show old limit and trigger "Context overflow" errors.

**Cause:** `contextTokens` is cached per-session when created. Config changes don't update existing sessions.

**Workaround:** Delete BOTH the session transcript AND the sessions.json entry:
```bash
# Delete the transcript file
rm ~/.openclaw/agents/main/sessions/<session-id>.jsonl

# Remove entry from sessions.json (CRITICAL - just deleting .jsonl is not enough!)
cd ~/.openclaw/agents/main/sessions
cat sessions.json | jq 'del(.["<session-key>"])' > sessions.json.new && mv sessions.json.new sessions.json

# Restart gateway
openclaw gateway restart
```

**Important:** Session key format is like `agent:main:telegram:dm:141413702` - check sessions.json to find the correct key.

### Session Transcript Corruption (Learned 2026-02-01)

**Problem:** Bot stops responding, and checking session transcript shows consecutive user messages without assistant responses.

**Cause:** LLM APIs require alternating user/assistant turns. Consecutive user messages (e.g., 5 user messages in a row) corrupt the session.

**Fix:** Same as context overflow - delete both .jsonl AND sessions.json entry, then restart gateway.

### Single-Provider Fallback Cascade Failure (Learned 2026-01-31)

**Problem:** All fallback models fail together when auth profile goes into cooldown.

**Cause:** If all models use the same provider (e.g., all `google-gemini-cli/*`) with one auth profile, a timeout on any model puts that profile in cooldown, blocking ALL models.

**Solution:**
1. Add a different provider as fallback (e.g., `anthropic/claude-sonnet-4`)
2. Add multiple auth profiles for the same provider: `openclaw models auth login --provider google-gemini-cli`

### Timeouts Treated as Rate Limits (Learned 2026-01-31)

**Problem:** API timeouts trigger auth profile cooldowns even when not actually rate-limited.

**Cause:** OpenClaw can't distinguish slow API from rate limiting, so treats all timeouts as "possible rate limit" with exponential backoff (1m → 5m → 25m → 1h).

**Mitigation:**
- Increase `timeoutSeconds` for large-context requests (300-600s for 100k+ tokens)
- Add multiple auth profiles for automatic rotation during cooldowns

### Manually Clear Cooldowns (Learned 2026-02-01)

**Problem:** All auth profiles in cooldown, bot can't respond. Error may show misleading "Context overflow" message.

**Solution:** Edit `~/.openclaw/agents/main/agent/auth-profiles.json` and remove cooldown data:

```bash
# Find usageStats section and remove:
# - cooldownUntil
# - errorCount (set to 0)
# - lastFailureAt
# - failureCounts

# Then restart gateway:
openclaw gateway restart
```

**What to remove from each profile in `usageStats`:**
```json
{
  "usageStats": {
    "google-gemini-cli:email@gmail.com": {
      "lastUsed": 1769922923662,
      "errorCount": 0  // Reset to 0, remove cooldownUntil/lastFailureAt/failureCounts
    }
  }
}
```

## Migration from Clawdbot to OpenClaw (Learned 2026-02-01)

As of version 2026.1.30, the package has been renamed from `clawdbot` to `openclaw`.

### Migration Steps

1. **Backup first:**
   ```bash
   cp -r ~/.clawdbot ~/.clawdbot.backup.$(date +%Y%m%d-%H%M%S)
   ```

2. **Stop old gateway:**
   ```bash
   systemctl --user stop clawdbot-gateway.service
   ```

3. **Install new package:**
   ```bash
   npm install -g openclaw@latest
   ```

4. **Run installer (handles migration automatically):**
   ```bash
   openclaw install
   # or: npx openclaw install
   ```
   The installer will:
   - Migrate `~/.clawdbot` → `~/.openclaw`
   - Create symlinks for backwards compatibility
   - Migrate `clawdbot.json` → `openclaw.json`
   - Install new `openclaw-gateway.service`

5. **Clean up old service:**
   ```bash
   systemctl --user disable --now clawdbot-gateway.service
   rm ~/.config/systemd/user/clawdbot-gateway.service
   systemctl --user daemon-reload
   ```

6. **PATH fix (if needed):**
   If `openclaw: command not found`, create symlink:
   ```bash
   ln -sf /home/<user>/.nvm/versions/node/v24.13.0/bin/openclaw ~/.local/bin/openclaw
   ```

### Post-Migration

- Run `openclaw configure` to re-authenticate any expired auth profiles
- Run `openclaw doctor` to verify health
- Old `clawdbot` command may still work via PATH, but use `openclaw` going forward

## Troubleshooting

### Run Diagnostics

```bash
openclaw doctor
openclaw doctor --fix
```

### Check Gateway Status

```bash
openclaw status
openclaw health
```

### Restart Gateway

```bash
openclaw gateway restart
```

**Note:** Top-level `openclaw restart` does not exist. Use `openclaw gateway restart`.

## Additional Resources

### Reference Files

Detailed documentation in `references/`:

- **`tools-core.md`** - exec tool, security modes, background processes
- **`tools-browser.md`** - browser automation, CDP, snapshots
- **`tools-sessions.md`** - sessions, sub-agents, multi-agent
- **`tools-automation.md`** - cron, webhooks, polling
- **`channels-messaging.md`** - WhatsApp, Telegram, Discord, Slack, Signal
- **`channels-apple.md`** - iMessage configuration
- **`configuration.md`** - moltbot.json, bootstrap files
- **`commands-reference.md`** - CLI commands
- **`skills-system.md`** - skills, ClawdHub
- **`agent-architecture.md`** - agents, routing, sandboxing
- **`memory-sessions.md`** - memory, session management
- **`baileys-direct-access.md`** - Direct WhatsApp/Baileys access for advanced operations

### Local Documentation

OpenClaw includes built-in local documentation accessible via CLI:

```bash
# View docs on any topic
openclaw docs <topic>

# Examples:
openclaw docs channels/whatsapp
openclaw docs concepts/agent-workspace
openclaw docs tools/exec
```

This is useful when working offline or when the remote docs site is slow.

### External Documentation

- **Official docs:** https://docs.openclaw.ai/
- **LLM reference:** https://docs.openclaw.ai/llms.txt
- **GitHub:** https://github.com/openclaw/openclaw

---

## Continuous Improvement

**CRITICAL:** This skill must be kept current. OpenClaw is under active development and changes frequently.

### The Learning Loop

```
Use OpenClaw → Discover something → Update this skill → Future sessions benefit
```

**If you don't write it down, future-you loses the knowledge.** This is the same principle as memory files.

### Workflow for Every OpenClaw Task

1. **Check current docs first** - WebFetch the relevant page from https://docs.openclaw.ai/ before relying solely on this skill
2. **Compare with this skill** - Note any differences between current docs and this skill
3. **Update immediately** - If the docs have new or different information:
   - Update the relevant reference file in `~/.claude/skills/openclaw-guide/references/`
   - Add new sections for newly discovered features
   - Correct any outdated or inaccurate information
   - Add practical examples from real usage
4. **Learn from errors** - If something doesn't work as documented here, fetch the latest docs and update

### What to Update

| You discovered... | Update... |
|-------------------|-----------|
| New config option | `references/configuration.md` |
| Channel behavior | `references/channels-messaging.md` |
| Tool behavior | `references/tools-*.md` |
| Agent/routing pattern | `references/agent-architecture.md` |
| Behavioral guideline | `references/agent-behavior.md` |
| Memory system change | `references/memory-sessions.md` |
| New workaround | Relevant reference + "Known Limitations" in SKILL.md |
| User correction | Whatever file was wrong |

### Reference Files

| File | Content |
|------|---------|
| `agent-architecture.md` | Agents, routing, sandboxing, multi-agent |
| `agent-behavior.md` | Memory habits, heartbeats, group chat etiquette |
| `baileys-direct-access.md` | WhatsApp/Baileys scripts |
| `channels-apple.md` | iMessage setup |
| `channels-messaging.md` | WhatsApp, Telegram, Discord, Slack, Signal |
| `commands-reference.md` | CLI commands |
| `configuration.md` | moltbot.json complete schema |
| `memory-sessions.md` | Memory system, sessions |
| `skills-system.md` | Skills, ClawdHub |
| `tools-automation.md` | Cron, webhooks, polling |
| `tools-browser.md` | Browser automation |
| `tools-core.md` | Exec tool, security |
| `tools-sessions.md` | Sessions, sub-agents |

### Remember

Every interaction with OpenClaw is an opportunity to improve this knowledge base. Treat discrepancies between this skill and official docs as bugs to fix immediately.

**Text > Brain. Write it down.**
