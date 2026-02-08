# CLI Commands Reference

## Command Overview

OpenClaw provides 50+ CLI commands for managing the gateway, channels, sessions, and tools.

## Core Commands

### Gateway Management

| Command | Description |
|---------|-------------|
| `openclaw gateway start` | Start the gateway daemon |
| `openclaw gateway stop` | Stop the gateway daemon |
| `openclaw gateway restart` | Restart the gateway (v2026.1.24-3 verified) |
| `openclaw status` | Check gateway status |
| `openclaw health` | Health check endpoint |

**Note (2026-01-31):** Top-level `openclaw restart` does not exist in v2026.1.24-3. Use `openclaw gateway restart` instead.

### Setup & Configuration

| Command | Description |
|---------|-------------|
| `openclaw onboard` | Run setup wizard |
| `openclaw setup` | Initialize configuration |
| `openclaw configure` | Edit configuration |
| `openclaw update` | Update to latest version |
| `openclaw uninstall` | Remove OpenClaw |

### Diagnostics

| Command | Description |
|---------|-------------|
| `openclaw doctor` | Run diagnostics |
| `openclaw doctor --repair` | Apply fixes, backup config |
| `openclaw doctor --deep` | Comprehensive checks |
| `openclaw logs` | View gateway logs |

## Channel Commands

### Authentication

```bash
# Login to channel
openclaw channels login
openclaw channels login --channel telegram
openclaw channels login --account secondary

# List channels
openclaw channels

# Channel status
openclaw channels status
```

### Pairing

```bash
# List pending approvals
openclaw pairing

# Approve sender
openclaw pairing approve whatsapp <code>
openclaw pairing approve telegram <code>

# Deny sender
openclaw pairing deny <code>
```

## Session Commands

### List Sessions

```bash
# Basic listing
openclaw sessions

# Active sessions (within N minutes)
openclaw sessions --active 120

# JSON output
openclaw sessions --json
```

### In-Chat Commands

| Command | Description |
|---------|-------------|
| `/status` | Show session status |
| `/context list` | List context items |
| `/new` | Start new session |
| `/reset` | Reset current session |

## Message Commands

Send messages, polls, and perform channel actions via CLI.

### Send Messages

```bash
# Send text message
openclaw message send --channel telegram --target @username --message "Hello"
openclaw message send --channel whatsapp --target +15555550123 --message "Hi"

# Send with media
openclaw message send --target +15555550123 --message "Photo" --media photo.jpg

# Telegram with inline buttons
openclaw message send --channel telegram --target @username \
  --message "Choose:" \
  --buttons '[[{"text":"Yes","callback_data":"yes"},{"text":"No","callback_data":"no"}]]'
```

### Polls (Telegram/Discord)

```bash
# Single-select poll
openclaw message poll --channel telegram --target @username \
  --poll-question "Favorite color?" \
  --poll-option "Red" --poll-option "Blue" --poll-option "Green"

# Multi-select poll (users can pick multiple)
openclaw message poll --channel telegram --target @username \
  --poll-question "Which apply?" \
  --poll-multi \
  --poll-option "Option A" --poll-option "Option B" --poll-option "Option C"
```

Poll options: 2-12 choices per poll, `--poll-multi` enables multiple selections.

### Other Actions

```bash
# React to message
openclaw message react --channel discord --target 123 --message-id 456 --emoji "✅"

# Read recent messages
openclaw message read --channel telegram --target @username --limit 10

# Edit/delete
openclaw message edit --channel telegram --target @username --message-id 123 --message "Updated"
openclaw message delete --channel telegram --target @username --message-id 123
```

## Cron Commands

```bash
# List cron jobs
openclaw cron

# Show job details
openclaw cron show <job-id>

# Edit job delivery
openclaw cron edit <job-id> --deliver --channel telegram --to "123456789"

# Disable delivery
openclaw cron edit <job-id> --no-deliver
```

## Browser Commands

### Navigation

```bash
# Open URL
openclaw browser open https://example.com

# Take snapshot
openclaw browser snapshot
openclaw browser snapshot --interactive
openclaw browser snapshot --mode role

# Screenshot
openclaw browser screenshot
openclaw browser screenshot --full-page
```

### Interaction

```bash
# Click element
openclaw browser click e12
openclaw browser click e12 --double

# Type text
openclaw browser type e23 "text"
openclaw browser type e23 "query" --submit

# Drag and drop
openclaw browser drag e10 e11
```

### State

```bash
# Cookies
openclaw browser cookies set name value --url "https://example.com"
openclaw browser cookies get --url "https://example.com"

# Geolocation
openclaw browser set geo 37.7749 -122.4194

# Timezone
openclaw browser set timezone America/New_York
```

## Agent Commands

```bash
# List agents
openclaw agents

# Agent info
openclaw agent <agent-id>

# Switch agent
openclaw agent use <agent-id>
```

## Memory Commands

```bash
# View memory
openclaw memory

# Search memory
openclaw memory search "query"
```

## Node Commands

```bash
# List nodes
openclaw nodes

# Node status
openclaw nodes status

# Pair node
openclaw nodes pair
```

## Skills Commands

```bash
# List skills
openclaw skills

# Skill info
openclaw skills show <skill-name>

# Enable/disable skill
openclaw skills enable <skill-name>
openclaw skills disable <skill-name>
```

## ClawdHub Commands

```bash
# Install skill
clawdhub install <skill-slug>

# Update all skills
clawdhub update --all

# Sync skills
clawdhub sync --all

# Search skills
clawdhub search "keyword"
```

## In-Chat Slash Commands

### Session Control

| Command | Description |
|---------|-------------|
| `/new` | Start new session |
| `/reset` | Reset session |
| `/status` | Show status |
| `/context list` | List context |

### Mode Control

| Command | Description |
|---------|-------------|
| `/elevated on` | Enable elevated mode |
| `/elevated full` | Full elevated (auto-approve) |
| `/elevated off` | Disable elevated |
| `/exec` | Toggle exec defaults |

### Sub-agents

| Command | Description |
|---------|-------------|
| `/subagents` | List sub-agents |
| `/subagents list` | List active |
| `/subagents stop <id>` | Stop sub-agent |
| `/subagents send <id> <msg>` | Send message |

### Process Control

| Command | Description |
|---------|-------------|
| `/process` | List background processes |
| `/process poll <id>` | Check process status |
| `/process send-keys <id>` | Send input |
| `/process submit <id>` | Submit command |

### Configuration

| Command | Description |
|---------|-------------|
| `/config` | Show configuration |
| `/activation` | Set activation mode |

## Debug Commands

```bash
# Enable debug mode
openclaw --debug

# Verbose logging
openclaw --verbose

# TUI interface
openclaw tui
```

## Useful Command Patterns

### Quick Health Check

```bash
openclaw status && openclaw health
```

### Reset Channel Auth

```bash
openclaw channels login --channel whatsapp --force
```

### Export Session Data

```bash
openclaw sessions --json > sessions.json
```

### Debug Gateway Issues

```bash
openclaw doctor --deep
openclaw logs --tail 100
```

---

## Continuous Improvement

**Always verify with current docs:** Before implementing, fetch the relevant page from https://docs.molt.bot/ to check for updates.


When using OpenClaw and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/openclaw-guide/references/commands-reference.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage
