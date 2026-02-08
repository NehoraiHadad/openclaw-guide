# Apple Channels - iMessage

## Overview

iMessage integration operates as an external CLI tool that spawns `imsg rpc` using JSON-RPC over stdio. Enables deterministic message routing with session isolation for group chats.

## Requirements

- macOS with signed-in Messages app
- Full Disk Access permissions for OpenClaw and `imsg`
- Automation permission for sending messages

## Setup

### Install imsg CLI

```bash
brew install steipete/tap/imsg
```

### Basic Configuration

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/usr/local/bin/imsg",
      dbPath: "/Users/<username>/Library/Messages/chat.db"
    }
  }
}
```

### Start Gateway

```bash
openclaw start
```

Approve system prompts when they appear.

## Advanced Setups

### Dedicated Bot Identity

Create separate Apple ID and macOS user for isolated bot messaging:

1. Create new macOS user
2. Sign in with dedicated Apple ID
3. Enable Remote Login (System Preferences > Sharing)
4. Configure SSH passwordless access
5. Point `cliPath` to SSH wrapper script

### SSH Wrapper Script

```bash
#!/bin/bash
ssh botuser@localhost /usr/local/bin/imsg "$@"
```

### Remote/SSH Variant

For running gateway on different machine:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "/path/to/ssh-wrapper.sh",
      remoteHost: "mac-mini.local"
    }
  }
}
```

Enables automatic SCP attachment retrieval from remote Mac.

### Tailscale Integration

Connect Linux gateway to Mac via Tailscale:

1. Install Tailscale on both machines
2. Configure SSH access over tailnet
3. Use SSH wrapper pointing to Mac's Tailscale IP

## Access Control

### Direct Messages

| Policy | Behavior |
|--------|----------|
| `pairing` | Approval codes for unknown senders (default) |
| `allowlist` | Only contacts in allowFrom |
| `open` | Accept all DMs |
| `disabled` | No direct messaging |

### Pairing Approval

```bash
openclaw pairing approve imessage <code>
```

Codes expire after 1 hour.

### Group Chats

| Policy | Behavior |
|--------|----------|
| `open` | Accept all groups |
| `allowlist` | Only groups in groupAllowFrom |
| `disabled` | Ignore groups (default) |

Configure group allowlist:
```json5
{
  channels: {
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["chat123", "chat456"]
    }
  }
}
```

### Mention Gating

```json5
{
  agents: {
    list: [{
      id: "main",
      groupChat: {
        mentionPatterns: ["@bot", "hey bot"]
      }
    }]
  }
}
```

## Message Handling

### Text Limits

Outbound text chunks to `textChunkLimit` (default: 4000 chars).

**Chunking modes:**
- `length` (default): Split at character limit
- `newline`: Split on paragraph boundaries first

```json5
{
  channels: {
    imessage: {
      textChunkLimit: 4000,
      chunkMode: "newline"
    }
  }
}
```

### Media Support

```json5
{
  channels: {
    imessage: {
      includeAttachments: true,
      mediaMaxMb: 16
    }
  }
}
```

## Session Isolation

| Chat Type | Session Key |
|-----------|-------------|
| DMs | Agent's main session (shared) |
| Groups | `agent:<id>:imessage:group:<id>` |
| Multi-participant | Configurable as groups |

## Routing & Addressing

### Address Formats

Prefer stable `chat_id` for delivery. Alternatives:

| Format | Example |
|--------|---------|
| `chat_id` | Stable identifier (preferred) |
| `chat_guid` | GUID format |
| `chat_identifier` | Chat identifier string |
| Handle | `+15551234567`, `email@example.com` |

### List Available Chats

```bash
imsg chats --limit 20
```

## Configuration Reference

| Option | Default | Description |
|--------|---------|-------------|
| `enabled` | false | Enable iMessage channel |
| `cliPath` | - | Path to imsg binary |
| `dbPath` | - | Path to chat.db |
| `dmPolicy` | pairing | DM access control |
| `groupPolicy` | disabled | Group access control |
| `historyLimit` | 50 | Group context messages |
| `dmHistoryLimit` | 100 | DM context messages |
| `includeAttachments` | false | Enable media ingestion |
| `textChunkLimit` | 4000 | Max chars per message |
| `chunkMode` | length | Chunking strategy |
| `configWrites` | true | Allow /config commands |
| `remoteHost` | - | Remote Mac hostname |

## Troubleshooting

### Permissions Issues

Check System Preferences > Security & Privacy:
- Full Disk Access: Terminal, OpenClaw, imsg
- Automation: Allow controlling Messages

### Not Receiving Messages

1. Verify imsg installation: `imsg version`
2. Check db path: `ls -la ~/Library/Messages/chat.db`
3. Test manually: `imsg chats --limit 5`

### Remote Setup Issues

1. Verify SSH access: `ssh botuser@localhost "imsg version"`
2. Check passwordless auth configured
3. Verify SCP access for attachments

---

## Continuous Improvement

**Always verify with current docs:** Before implementing, fetch the relevant page from https://docs.molt.bot/ to check for updates.


When using OpenClaw and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/openclaw-guide/references/channels-apple.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage
