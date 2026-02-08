# Memory and Session Management

## Memory System Overview

OpenClaw stores information as **plain Markdown files in the agent workspace**, treating files as the source of truth rather than RAM.

## Memory Layers

### Daily Logs

Location: `memory/YYYY-MM-DD.md`

- Append-only records loaded at session start
- Today's and yesterday's entries loaded
- Use for transient notes and session-specific info

### Long-term Memory

Location: `MEMORY.md`

- Curated durable facts
- Loaded only in private sessions (never in groups)
- Use for decisions, preferences, lasting facts

## Writing Guidelines

| Content Type | Location |
|--------------|----------|
| Decisions, preferences | `MEMORY.md` |
| Lasting facts | `MEMORY.md` |
| Transient notes | `memory/YYYY-MM-DD.md` |
| Session-specific info | Daily logs |

**Best practice:** Explicitly request the bot to write important information to disk.

## Automatic Memory Flush

Before context compaction, OpenClaw triggers a silent agentic turn to write durable memories.

**Activation conditions:**
- Context nears threshold: `contextWindow - reserveTokensFloor - softThresholdTokens`
- Prompts allow `NO_REPLY` responses to remain silent
- Occurs once per compaction cycle
- Requires writable workspace (skipped if read-only)

**Configuration:**
```json5
{
  memory: {
    autoFlush: {
      enabled: true,
      softThresholdTokens: 1000
    }
  }
}
```

## Vector Memory Search

The system indexes Markdown files for semantic searching.

### Indexed Files

- `MEMORY.md`
- `memory/*.md`
- Optional extra paths via config

### Provider Selection (Auto Order)

1. Local mode (if configured model exists)
2. OpenAI (if API key available)
3. Gemini (if API key available)
4. Disabled otherwise

### Configuration

```json5
{
  memory: {
    vector: {
      enabled: true,
      provider: "openai",  // or "gemini", "local"
      model: "text-embedding-3-small"
    }
  }
}
```

### Search Tools

| Tool | Description |
|------|-------------|
| `memory_search` | Returns semantically matching snippets (~400-token chunks with 80-token overlap) |
| `memory_get` | Reads specific memory files by path |

### Embedding Storage

SQLite at `~/.openclaw/memory/<agentId>.sqlite`

Optional sqlite-vec acceleration for faster vector queries.

## Hybrid Search (BM25 + Vector)

Combines vector similarity (semantic matches) with BM25 keyword relevance (exact tokens).

**Process:**
1. Retrieve candidate pools from both signals
2. Convert BM25 ranks to 0..1 scores
3. Weight results (default: 0.7 vector, 0.3 text)
4. Return unified results

**Configuration:**
```json5
{
  memory: {
    vector: {
      hybrid: {
        enabled: true,
        vectorWeight: 0.7,
        textWeight: 0.3,
        candidateMultiplier: 2
      }
    }
  }
}
```

## Advanced Features

### Batch Indexing (OpenAI/Gemini)

Submits multiple embedding requests asynchronously for faster large-scale indexing.

```json5
{
  memory: {
    vector: {
      batch: {
        concurrency: 2
      }
    }
  }
}
```

### Embedding Cache

Prevents re-embedding unchanged text during frequent updates.

```json5
{
  memory: {
    vector: {
      cache: {
        maxEntries: 50000
      }
    }
  }
}
```

### Session Memory (Experimental)

Optionally indexes session transcripts alongside memory files.

```json5
{
  experimental: {
    sessionMemory: true
  }
}
```

### Custom Endpoints

Supports OpenAI-compatible endpoints:

```json5
{
  memory: {
    vector: {
      provider: "custom",
      baseUrl: "https://api.example.com/v1",
      apiKey: "...",
      headers: { "X-Custom": "value" }
    }
  }
}
```

### Local Model

Default local model (~0.6 GB) auto-downloads on first use.

Requires `pnpm approve-builds` for native builds.

---

## Session Management

### Session Key Formats

| Chat Type | Format |
|-----------|--------|
| DMs (main scope) | `agent:<agentId>:<mainKey>` |
| DMs (per-peer) | `agent:<agentId>:dm:<peerId>` |
| DMs (per-channel-peer) | `agent:<agentId>:<channel>:dm:<peerId>` |
| DMs (per-account-channel-peer) | `agent:<agentId>:<channel>:<accountId>:dm:<peerId>` |
| Groups | `agent:<agentId>:<channel>:group:<id>` |
| Telegram topics | `...group:<id>:topic:<threadId>` |
| Cron jobs | `cron:<job.id>` |
| Webhooks | `hook:<uuid>` |
| Sub-agents | `agent:<agentId>:subagent:<uuid>` |
| Node runs | `node-<nodeId>` |

### Session Scope Settings

Control DM isolation with `sessions.dmScope`:

| Value | Behavior |
|-------|----------|
| `main` | Continuity across channels (default) |
| `per-peer` | Isolated per contact |
| `per-channel-peer` | Isolated per channel+contact |
| `per-account-channel-peer` | Multi-account isolation |

### Identity Links

Map users across platforms:

```json5
{
  sessions: {
    identityLinks: {
      "john": ["+15551234567", "discord:123456789"]
    }
  }
}
```

Same person shares session even with `per-peer` isolation.

### Session Storage

- **Location:** `~/.openclaw/agents/<agentId>/sessions/`
- **Transcripts:** `<SessionId>.jsonl`
- **Store:** `sessions.json` with `sessionKey → {sessionId, updatedAt, displayName, channel, subject, room, space}`
- **Authority:** Gateway is source of truth

### Reset Policies

**Daily Reset:**
```json5
{
  sessions: {
    dailyResetHour: 4  // 4 AM gateway local time
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

**Per-Channel Overrides:** `resetByChannel` takes precedence

**Manual Triggers:** `/new`, `/reset`, custom `resetTriggers`

### Pruning & Compaction

- OpenClaw trims old tool results from in-memory context before LLM calls
- Complete JSONL transcript history preserved
- Pre-compaction memory flush writes durable notes at limits

### Session Origin Metadata

Each entry records: `label`, `provider`, `from`/`to`, `accountId`, `threadId`

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

## Monitoring

```bash
openclaw sessions
openclaw sessions --active 120
openclaw sessions --json
```

In-chat: `/status`, `/context list`

## Best Practices

### Memory

1. Use `MEMORY.md` for durable, cross-session facts
2. Use daily logs for transient notes
3. Explicitly request important info be saved
4. Review and curate `MEMORY.md` periodically

### Sessions

1. Use appropriate `dmScope` for your use case
2. Set up identity links for multi-platform users
3. Configure reset policies per chat type
4. Monitor sessions for unusual patterns

### Performance

1. Enable vector search for large memory sets
2. Configure appropriate chunk sizes
3. Use embedding cache for frequent indexing
4. Prune old sessions periodically

---

## Continuous Improvement

**Always verify with current docs:** Before implementing, fetch the relevant page from https://docs.molt.bot/concepts/memory to check for updates.

When using OpenClaw and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/openclaw-guide/references/memory-sessions.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage
