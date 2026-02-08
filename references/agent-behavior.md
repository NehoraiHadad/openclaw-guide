# Agent Behavior & Bootstrap System

This reference documents how OpenClaw's bootstrap and behavior system works — the platform mechanics, not prescriptive rules.

**Source:** Official docs at `~/.nvm/.../openclaw/docs/` and templates at `.../docs/reference/templates/`

## Bootstrap File System

### How It Works

On session start, OpenClaw injects workspace files into the context. This is the agent's "memory" and "personality" system.

**Injection order:** AGENTS.md → SOUL.md → USER.md → IDENTITY.md → TOOLS.md → (memory files)

**Key behaviors:**
- Missing files: OpenClaw injects a "missing file" marker and continues
- Large files: Truncated at `agents.defaults.bootstrapMaxChars` (default: 20000)
- Skip bootstrap: Set `agents.defaults.skipBootstrap: true` to manage files yourself

### File Purposes (Platform-Defined)

| File | Purpose | When Loaded |
|------|---------|-------------|
| `AGENTS.md` | Operating instructions, memory rules, behavior guidelines | Every session |
| `SOUL.md` | Persona, tone, boundaries | Every session |
| `USER.md` | User profile and preferences | Every session |
| `IDENTITY.md` | Agent name, vibe, emoji | Every session |
| `TOOLS.md` | Local tool notes and conventions | Every session |
| `HEARTBEAT.md` | Checklist for heartbeat runs | On heartbeat only |
| `BOOT.md` | Startup checklist on gateway restart | On boot only |
| `BOOTSTRAP.md` | One-time first-run ritual | First run only (delete after) |
| `MEMORY.md` | Curated long-term memory | Main session only (not groups) |
| `memory/YYYY-MM-DD.md` | Daily logs | Today + yesterday |

### Default Templates

OpenClaw ships with default templates at:
```
~/.nvm/.../openclaw/docs/reference/templates/
├── AGENTS.md       # Full behavioral framework
├── AGENTS.dev.md   # Developer-focused variant
├── SOUL.md         # Default persona template
├── SOUL.dev.md     # Developer-focused variant
├── USER.md         # User profile template
├── TOOLS.md        # Tool notes template
├── IDENTITY.md     # Identity template
├── BOOTSTRAP.md    # First-run ritual
├── HEARTBEAT.md    # Heartbeat checklist template
└── BOOT.md         # Boot checklist template
```

**To use defaults:**
```bash
cp docs/reference/templates/AGENTS.md ~/clawd/AGENTS.md
cp docs/reference/templates/SOUL.md ~/clawd/SOUL.md
# etc.
```

Or run `openclaw setup` to seed missing files.

---

## Memory System Mechanics

### How Memory Works

Memory is **plain Markdown in the workspace**. The model only "remembers" what gets written to disk.

**Two layers:**
1. `memory/YYYY-MM-DD.md` — Daily append-only logs
2. `MEMORY.md` — Curated long-term memory

### Security Isolation

`MEMORY.md` is **only loaded in main session** (direct chats with the human). It is NOT loaded in:
- Group chats
- Discord channels
- Sessions with other people

This prevents personal context from leaking to shared contexts.

### Automatic Memory Flush

Before context compaction, OpenClaw triggers a silent turn to write durable memories.

**Config:**
```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      }
    }
  }
}
```

**Activation:** When token estimate crosses `contextWindow - reserveTokensFloor - softThresholdTokens`

**Note:** Flush is skipped if workspace is read-only (`workspaceAccess: "ro"` or `"none"`).

### Vector Memory Search

OpenClaw can build a semantic index over memory files.

**Default behavior:**
- Indexes `MEMORY.md` + `memory/**/*.md`
- Auto-selects provider: local → openai → gemini
- Stores in `~/.openclaw/memory/<agentId>.sqlite`

**Tools provided:**
- `memory_search` — Semantic search across memory
- `memory_get` — Read specific memory file

**Config:**
```json5
{
  agents: {
    defaults: {
      memorySearch: {
        provider: "openai",  // or "gemini", "local"
        model: "text-embedding-3-small",
        query: {
          hybrid: { enabled: true, vectorWeight: 0.7, textWeight: 0.3 }
        }
      }
    }
  }
}
```

---

## Heartbeat System

### How Heartbeats Work

Heartbeats are periodic polls that give the agent a chance to do proactive work.

**Default heartbeat prompt:**
```
Read HEARTBEAT.md if it exists (workspace context). Follow it strictly.
Do not infer or repeat old tasks from prior chats.
If nothing needs attention, reply HEARTBEAT_OK.
```

**Config:**
```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "3600",              // seconds or duration string
        model: "anthropic/claude-sonnet-4",
        session: "heartbeat",
        target: "whatsapp",
        to: "+15551234567"
      }
    }
  }
}
```

### HEARTBEAT.md

Optional checklist file in workspace. Keep it small to limit token burn.

The agent reads this file on heartbeat and acts on it. Users can customize what the agent checks.

### Heartbeat vs Cron

| Feature | Heartbeat | Cron |
|---------|-----------|------|
| Timing | Approximate (~30 min drift OK) | Exact ("9:00 AM sharp") |
| Context | Has conversation context | Isolated session |
| Model | Uses default model | Can specify different model |
| Use case | Batch multiple checks | Precise schedules, one-shot reminders |

---

## Group Chat Behavior Framework

### Mention Detection

**Agent-level:** Configure `groupChat.mentionPatterns` on the agent:
```json5
{
  agents: {
    list: [{
      id: "main",
      groupChat: {
        mentionPatterns: ["@clawd", "hey clawd", "להביא", "מחר"]
      }
    }]
  }
}
```

**Channel-level:** Configure `requireMention` per group:
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

### Silent Monitor Pattern

For agents that receive messages but shouldn't respond in the group:

1. Set `requireMention: false` on channel (receive all messages)
2. Set `mentionPatterns` on agent (filter by keywords)
3. Restrict tools: `deny: ["whatsapp_send"]`
4. Allow only `sessions_send` or `telegram_send` for alerts

---

## Platform Formatting Constraints

These are technical limitations of messaging platforms:

| Platform | Constraint |
|----------|------------|
| Discord | Max 2000 chars per message |
| WhatsApp | No markdown headers render |
| Discord/WhatsApp | Markdown tables don't render well |
| Discord | Multiple links trigger embed flood |

The default AGENTS.md template includes guidance for agents to handle these.

---

## Workspace Backup

Recommended: Put workspace in a **private** git repo.

```bash
cd ~/clawd
git init
git add AGENTS.md SOUL.md TOOLS.md IDENTITY.md USER.md HEARTBEAT.md memory/
git commit -m "Add agent workspace"
```

**Do not commit:**
- API keys, OAuth tokens, passwords
- Anything under `~/.openclaw/`
- Sensitive chat dumps

---

## Configuration Reference

### Bootstrap-Related Config

```json5
{
  agents: {
    defaults: {
      workspace: "~/clawd",
      skipBootstrap: false,
      bootstrapMaxChars: 20000
    }
  }
}
```

### Memory-Related Config

```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000
        }
      },
      memorySearch: {
        provider: "openai",
        model: "text-embedding-3-small"
      }
    }
  }
}
```

---

## Continuous Improvement

**Update this reference when:**
- Platform mechanics change (new config options, new behavior)
- Official docs differ from what's here
- New bootstrap files are added

**Do NOT add:**
- Specific behavioral rules (those belong in workspace templates)
- User-specific configurations
- Opinions about what agents should do

This reference documents **how the system works**, not **what to put in it**.

**Source paths:**
- Bundled docs: `~/.nvm/.../openclaw/docs/`
- Templates: `~/.nvm/.../openclaw/docs/reference/templates/`
- Online: https://docs.openclaw.ai/ (redirects from docs.molt.bot)
