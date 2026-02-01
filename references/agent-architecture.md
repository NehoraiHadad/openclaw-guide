# Agent Architecture

## Overview

Moltbot operates a single embedded agent runtime derived from p-mono. Session management and tool integration are handled independently by Moltbot.

## Agent Components

Each agent maintains:
- **Workspace:** Working directory for tools and context
- **State directory:** `agentDir` for agent-specific state
- **Session store:** Conversation transcripts
- **Bootstrap files:** Context injection on first turn

## Workspace Structure

### Mandatory Workspace

Set via `agents.defaults.workspace`. Serves as the sole working directory for all tools and context.

```json5
{
  agents: {
    defaults: {
      workspace: "~/workspace"
    }
  }
}
```

### Sandboxed Workspaces

Override with per-session workspaces:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        workspaceRoot: "~/sandboxed-workspaces"
      }
    }
  }
}
```

## Session Storage

Transcripts stored as JSONL:
```
~/.clawdbot/agents/<agentId>/sessions/<SessionId>.jsonl
```

Session IDs are stable, Moltbot-assigned identifiers.

## Multi-Agent Setup

### Agent Definition

```json5
{
  agents: {
    defaults: {
      workspace: "~/default-workspace",
      model: "anthropic/claude-opus-4-5"
    },
    list: [
      {
        id: "main",
        workspace: "~/main-workspace"
      },
      {
        id: "research",
        workspace: "~/research-workspace",
        model: "anthropic/claude-sonnet-4"
      },
      {
        id: "untrusted",
        workspace: "~/untrusted-workspace",
        sandbox: { mode: "all" }
      }
    ]
  }
}
```

### Agent Properties

| Property | Description |
|----------|-------------|
| `id` | Unique agent identifier |
| `workspace` | Agent workspace directory |
| `model` | LLM model override |
| `sandbox` | Sandbox settings |
| `tools` | Per-agent tool restrictions |

## Routing Architecture

Moltbot uses **deterministic, most-specific-wins routing**.

### Routing Priority

1. **Direct peer matches:** DM/group/channel ID
2. **Guild ID:** Discord servers
3. **Team ID:** Slack workspaces
4. **Account ID matching:** Multi-account setups
5. **Channel-wide fallback:** Channel default agent
6. **Default agent:** Global fallback

### Binding Configuration

**IMPORTANT:** `bindings` is a **top-level key**, NOT under `agents`.

```json5
{
  // bindings is at the ROOT level of moltbot.json
  bindings: [
    // Route WhatsApp channel to everyday agent
    {
      agentId: "chat",
      match: { channel: "whatsapp" }
    },
    // Route Telegram to opus agent
    {
      agentId: "opus",
      match: { channel: "telegram" }
    },
    // Route specific WhatsApp group to monitor agent
    {
      agentId: "monitor-school",
      match: {
        channel: "whatsapp",
        accountId: "personal",
        peer: {
          kind: "group",
          id: "120363000000000001@g.us"
        }
      }
    },
    // Route specific DM to premium agent
    {
      agentId: "premium",
      match: {
        channel: "whatsapp",
        peer: {
          kind: "dm",
          id: "+15551234567"
        }
      }
    }
  ],

  // Agent-to-agent communication (also at tools level)
  tools: {
    agentToAgent: {
      enabled: true,
      allow: ["monitor-active", "monitor-school"]
    }
  }
}
```

### Match Properties

| Property | Description |
|----------|-------------|
| `channel` | Channel name: `whatsapp`, `telegram`, `discord`, `slack` |
| `accountId` | Multi-account setups (e.g., `"personal"`, `"work"`) |
| `peer.kind` | `"dm"` or `"group"` |
| `peer.id` | WhatsApp JID (`...@g.us`), phone number, or channel ID |

### Key Principles

- Bindings route inbound messages to agents
- Peer-specific bindings always override channel-wide rules
- Each agent maintains separate credentials
- Explicit file copying needed for credential sharing

## Sandbox Configuration

### Sandbox Modes

| Mode | Behavior |
|------|----------|
| `off` | No sandboxing (default) |
| `non-main` | Sandbox non-main sessions |
| `all` | Sandbox all sessions |

### Per-Agent Sandbox

```json5
{
  agents: {
    list: [
      {
        id: "untrusted",
        sandbox: {
          mode: "all",
          workspaceAccess: "read-only"  // none|read-only|read-write
        }
      }
    ]
  }
}
```

### Docker Setup Commands

One-time initialization for containerized agents:

```json5
{
  agents: {
    list: [
      {
        id: "sandboxed",
        sandbox: {
          mode: "all",
          setupCommand: "apt-get update && apt-get install -y jq"
        }
      }
    ]
  }
}
```

### Security Features

Docker sandbox hardening includes:
- Network isolation
- Read-only root filesystem
- Capability dropping
- Memory/CPU limits
- Optional seccomp/AppArmor profiles

## Tool Restrictions

### Per-Agent Tool Policy

```json5
{
  agents: {
    list: [
      {
        id: "restricted",
        tools: {
          exec: {
            security: "allowlist",
            allowlist: ["/usr/bin/git", "/usr/bin/npm"]
          },
          browser: {
            enabled: false
          }
        }
      }
    ]
  }
}
```

### Elevated Mode Restrictions

**Important:** `tools.elevated` remains global and sender-based, not configurable per agent.

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: ["+15551234567"]  // Sender-based, applies to all agents
    }
  }
}
```

## Agent Collaboration

### Inter-Agent Messaging

Agent-to-agent messaging is disabled by default. When enabled, requires explicit allowlisting:

```json5
{
  agents: {
    collaboration: {
      enabled: true,
      allowlist: ["main", "research"]  // Only these agents can message each other
    }
  }
}
```

### Isolation Guarantees

- Each agent has separate:
  - Workspace
  - State directory
  - Session store
  - Authentication profiles
- Credentials don't share automatically
- Requires explicit file copying for credential sharing

## Message Queue Handling

### Queue Modes

| Mode | Behavior |
|------|----------|
| `steer` | Inject messages mid-run (skip remaining tool calls) |
| `followup` | Queue until turn completion |
| `collect` | Queue until turn completion |

### Configuration

```json5
{
  agents: {
    defaults: {
      queueMode: "followup"
    }
  }
}
```

## Group Chat Settings

### Mention Patterns (AGENT LEVEL)

**IMPORTANT:** `mentionPatterns` belongs on the **agent** (`agents.list[].groupChat.mentionPatterns`), NOT on channel groups. Channel groups only support `requireMention: true/false`.

```json5
{
  agents: {
    list: [
      {
        id: "monitor-school",
        groupChat: {
          // Hebrew trigger patterns - agent only wakes on these keywords
          mentionPatterns: ["להביא", "מחר", "שיעורי בית", "טיול", "ביטול", "תשלום"]
        },
        tools: {
          allow: ["sessions_send"],
          deny: ["whatsapp_send", "telegram_send"]
        }
      }
    ]
  }
}
```

### Per-Channel Group Settings

Channel groups only support `requireMention`, NOT `mentionPatterns`:

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        personal: {
          groups: {
            "*": { "requireMention": true },           // default: need @mention
            "120363...@g.us": { "requireMention": false }  // this group: all messages
          }
        }
      }
    },
    telegram: {
      groups: {
        "*": { requireMention: true },
        "-1001234567": { requireMention: false }
      }
    }
  }
}
```

### Silent Monitor Pattern

For a "silent" agent that monitors groups without responding in the group:

1. Set `requireMention: false` on channel groups (receive all messages)
2. Set `groupChat.mentionPatterns` on agent (filter by keywords)
3. Restrict tools: `deny: ["whatsapp_send"]` (can't respond in group)
4. Allow only `sessions_send` or `telegram_send` for alerts

```json5
{
  agents: {
    list: [{
      id: "monitor-school",
      groupChat: {
        mentionPatterns: ["להביא", "מחר", "תשלום", "טיול"]
      },
      tools: {
        allow: ["sessions_send", "telegram_send"],
        deny: ["whatsapp_send"]
      }
    }]
  },
  bindings: [{
    agentId: "monitor-school",
    match: { channel: "whatsapp", accountId: "personal", peer: { kind: "group", id: "...@g.us" } }
  }],
  channels: {
    whatsapp: {
      accounts: {
        personal: {
          groups: {
            "...@g.us": { "requireMention": false }
          }
        }
      }
    }
  }
}
```

## Common Patterns

### Fast Research Agent

```json5
{
  agents: {
    list: [
      {
        id: "research",
        model: "anthropic/claude-sonnet-4",
        workspace: "~/research"
      }
    ],
    bindings: {
      "telegram:group:-1001234567": "research"
    }
  }
}
```

### Isolated Untrusted Agent

```json5
{
  agents: {
    list: [
      {
        id: "public",
        sandbox: { mode: "all" },
        tools: {
          exec: { security: "deny" },
          browser: { enabled: false }
        }
      }
    ],
    bindings: {
      "discord:channel:publicchannel": "public"
    }
  }
}
```

### Premium Agent for Specific User

```json5
{
  agents: {
    list: [
      {
        id: "premium",
        model: "anthropic/claude-opus-4-5"
      }
    ],
    bindings: {
      "whatsapp:+15551234567": "premium"
    }
  }
}
```

---

## Local Documentation Reference

The official multi-agent documentation is bundled with clawdbot at:
```
~/.nvm/versions/node/<version>/lib/node_modules/clawdbot/docs/concepts/multi-agent.md
```

Key examples from official docs:
- Two WhatsApps → two agents
- WhatsApp daily chat + Telegram deep work
- Same channel, one peer to Opus
- Family agent bound to a WhatsApp group (with tool restrictions)

---

## Continuous Improvement

**Always verify with current docs:** Check the bundled docs at the clawdbot installation path, or fetch from https://docs.molt.bot/ to check for updates.


When using Clawdbot and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/clawdbot-guide/references/agent-architecture.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage
