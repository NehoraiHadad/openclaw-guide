# Sessions and Sub-agents

## Sub-agents Overview

Sub-agents are background agent runs spawned from an existing agent session. They operate in isolated sessions (`agent:<agentId>:subagent:<uuid>`) and report results back to the requester channel.

**Key constraint:** Sub-agents cannot spawn other sub-agents (no nesting).

## Spawning Sub-agents

### Via Tool (`sessions_spawn`)

```json
{
  "tool": "sessions_spawn",
  "input": {
    "task": "Research Node.js security vulnerabilities",
    "label": "Security research",
    "model": "anthropic/claude-sonnet-4",
    "thinking": "medium",
    "runTimeoutSeconds": 300,
    "cleanup": "keep"
  }
}
```

**Parameters:**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `task` | Yes | Work to execute |
| `label` | No | Descriptive name |
| `agentId` | No | Target different agent (if allowed) |
| `model` | No | Override default model |
| `thinking` | No | Override thinking level |
| `runTimeoutSeconds` | No | Abort after N seconds (default: 0) |
| `cleanup` | No | `"delete"` or `"keep"` (default) |

Returns immediately: `{ status: "accepted", runId, childSessionKey }`

### Via Slash Command

| Command | Description |
|---------|-------------|
| `/subagents list` | Inspect current subagents |
| `/subagents stop <id\|#\|all>` | Terminate runs |
| `/subagents log <id\|#> [limit] [tools]` | View execution logs |
| `/subagents info <id\|#>` | Display metadata |
| `/subagents send <id\|#> <message>` | Communicate with running subagent |

## Configuration

### Model Selection

Resolution priority (highest to lowest):
1. `sessions_spawn.model` parameter
2. `agents.list[].subagents.model` (per-agent)
3. `agents.defaults.subagents.model` (global)
4. Caller's model

```json5
{
  agents: {
    defaults: {
      subagents: {
        model: "anthropic/claude-sonnet-4",
        maxConcurrent: 8,
        archiveAfterMinutes: 60
      }
    }
  }
}
```

### Agent Targeting

Control which agents can be spawned:

```json5
{
  agents: {
    list: [{
      id: "main",
      subagents: {
        allowAgents: ["research", "ops"]  // or ["*"] for any
      }
    }]
  }
}
```

Default restricts to the requester only.

### Tool Restrictions

By default, subagents receive all tools **except** session-related ones:

**Denied:** `sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`

Customize:
```json5
{
  tools: {
    subagents: {
      tools: {
        deny: ["gateway", "cron"],
        allow: ["read", "exec", "process"]  // allow-only mode
      }
    }
  }
}
```

## Announcement System

Sub-agents automatically report results back via announce step:

- Reply of exactly `"ANNOUNCE_SKIP"` suppresses posting
- Results posted as follow-up messages with:
  - `Status:` success/error/timeout/unknown
  - `Result:` summary from announce step
  - `Notes:` errors and context
  - Runtime, token usage, estimated cost, transcript paths
- Thread/topic routing preserved (Slack, Telegram, Matrix)

## Auto-archiving & Cleanup

- Sessions auto-archive after `archiveAfterMinutes` (default: 60)
- Archived transcripts renamed: `*.deleted.<timestamp>`
- Use `cleanup: "delete"` for immediate archival

## Context Injection

Minimal: only `AGENTS.md` and `TOOLS.md` injected (not full workspace context).

---

## Session Management

### Session Key Formats

| Chat Type | Format |
|-----------|--------|
| DMs (main) | `agent:<agentId>:<mainKey>` |
| DMs (per-peer) | `agent:<agentId>:dm:<peerId>` |
| DMs (per-channel-peer) | `agent:<agentId>:<channel>:dm:<peerId>` |
| Groups | `agent:<agentId>:<channel>:group:<id>` |
| Telegram topics | `...group:<id>:topic:<threadId>` |
| Cron jobs | `cron:<job.id>` |
| Webhooks | `hook:<uuid>` |
| Sub-agents | `agent:<agentId>:subagent:<uuid>` |

### Session Scope Settings

Control DM session isolation with `dmScope`:

| Value | Behavior |
|-------|----------|
| `main` | Continuity across channels (default) |
| `per-peer` | Isolated per contact |
| `per-channel-peer` | Isolated per channel+contact |
| `per-account-channel-peer` | Multi-account isolation |

```json5
{
  sessions: {
    dmScope: "per-peer"
  }
}
```

### Identity Links

Map users across platforms for shared sessions:

```json5
{
  sessions: {
    identityLinks: {
      "john": ["+15551234567", "discord:123456789", "telegram:987654321"]
    }
  }
}
```

Same person shares session even with `per-peer` isolation.

### Session Storage

- Location: `~/.openclaw/agents/<agentId>/sessions/`
- Transcripts: `<SessionId>.jsonl`
- Store structure: `sessions.json` with map of `sessionKey → {sessionId, updatedAt, ...}`
- Gateway is source of truth

### Reset Policies

**Daily Reset:**
Default 4:00 AM gateway local time.

```json5
{
  sessions: {
    dailyResetHour: 4
  }
}
```

**Idle Reset:**
```json5
{
  sessions: {
    idleMinutes: 120
  }
}
```

**Per-Type Overrides:**
```json5
{
  sessions: {
    resetByType: {
      dm: { dailyResetHour: 4 },
      group: { idleMinutes: 120, dailyResetHour: null }
    }
  }
}
```

**Manual Triggers:** `/new`, `/reset`, plus custom `resetTriggers`

### Pruning & Compaction

- OpenClaw trims old tool results from in-memory context before LLM calls
- Complete JSONL transcript preserved
- Pre-compaction memory flush writes durable notes when near auto-compaction

## Monitoring

```bash
openclaw sessions
openclaw sessions --active 120
openclaw sessions --json
```

In-chat: `/status`, `/context list`

### Send Policy

Block delivery for specific session types:

```json5
{
  sessions: {
    sendPolicy: {
      blockTypes: ["group"]
    }
  }
}
```

Runtime override: `/send on|off|inherit`

---

## Continuous Improvement

**Always verify with current docs:** Before implementing, fetch the relevant page from https://docs.molt.bot/tools/subagents to check for updates.

When using OpenClaw and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/openclaw-guide/references/tools-sessions.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage
