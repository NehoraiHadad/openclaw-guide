# Messaging Channel Configuration

## WhatsApp

WhatsApp integration uses Baileys library for web sessions.

### Setup

1. Configure in `~/.clawdbot/moltbot.json`
2. Run `moltbot channels login`
3. Scan QR code with WhatsApp → Linked Devices

### Phone Number Options

**Dedicated Number (Recommended):**
- Separate phone/eSIM for Moltbot only
- Ideal: spare Android + eSIM on Wi-Fi/power
- WhatsApp Business can coexist on same device
- Cleanest routing, no self-chat quirks

**Personal Number:**
- Enable `selfChatMode: true`
- Message yourself for testing
- Must provide your own number during setup

**Avoid:** TextNow, Google Voice, free SMS services (aggressively blocked)

### Complete Configuration

```json5
{
  channels: {
    whatsapp: {
      enabled: true,
      dmPolicy: "pairing",      // pairing|allowlist|open|disabled
      allowFrom: ["+15551234567"],
      selfChatMode: false,
      sendReadReceipts: true,
      textChunkLimit: 4000,
      chunkMode: "length",      // length|newline
      mediaMaxMb: 50,           // inbound cap
      historyLimit: 50,         // group context messages
      dmHistoryLimit: 100,      // DM history (user turns)
      configWrites: true,       // allow /config commands
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions"       // always|mentions|never
      },
      groupPolicy: "allowlist", // open|disabled|allowlist
      groupAllowFrom: ["+15551234567"],
      groups: {
        "123456789@g.us": {
          allowFrom: ["+15551234567"],
          mention: true
        }
      }
    }
  }
}
```

### DM Policy Options

| Policy | Behavior |
|--------|----------|
| `pairing` | Unknown senders get code; approve via `moltbot pairing approve whatsapp <code>` |
| `allowlist` | Only numbers in `allowFrom` |
| `open` | Requires `allowFrom: ["*"]` |
| `disabled` | No direct messaging |

**Note:** Linked WhatsApp number is implicitly trusted.

### Self-Chat Mode Behaviors

When `selfChatMode: true`:
- Outbound DMs never trigger pairing replies
- Auto read receipts skipped
- Response prefix defaults to `[{identity.name}]`

### Group Configuration

| Setting | Description |
|---------|-------------|
| `groupPolicy` | `open`, `disabled`, `allowlist` (default) |
| `mention` activation | `true` (default) requires @mention or regex |
| `always` activation | Processes all messages |
| `historyLimit` | Recent unprocessed messages injected (default 50) |

Session key format: `agent:<agentId>:whatsapp:group:<jid>`

### Policy Behavior Details (IMPORTANT)

**Critical discovery:** The policy settings affect BOTH agent response AND message logging:

| Policy Setting | Messages Logged? | Agent Responds? | Use Case |
|----------------|------------------|-----------------|----------|
| `groupPolicy: "open"` | Yes | Yes (on mention) | Active bot in groups |
| `groupPolicy: "allowlist"` + groups configured | Yes | Yes (on mention) | Controlled group access |
| `groupPolicy: "allowlist"` + empty/no groups | **NO** | No | Blocks everything |
| `groupPolicy: "disabled"` | **NO** | No | Complete block |
| `dmPolicy: "disabled"` | **NO** | No | Ignores DMs entirely |

**Passive Monitoring Limitation:** There is NO built-in way to receive/log messages without the agent responding. The options are:
- `open` = messages logged, but agent wakes up (uses tokens)
- `allowlist` empty = messages NOT logged at all
- `disabled` = messages ignored entirely

**Workaround for passive monitoring:** See `references/baileys-direct-access.md` for direct Baileys access methods.

### Mention Patterns

Control what triggers the agent in groups via `agents.list[].groupChat.mentionPatterns`:

```json5
{
  agents: {
    list: [{
      id: "main",
      groupChat: {
        mentionPatterns: [
          "@?clawdbot",      // matches @clawdbot or clawdbot
          "\\+?15555550123"  // matches phone number
        ]
      }
    }]
  }
}
```

**Note:** Even with custom `mentionPatterns`, the canonical WhatsApp @-mention (tapping contact) triggers via `mentionedJids` and cannot be disabled.

### Acknowledgment Reactions

```json5
{
  ackReaction: {
    emoji: "👀",      // or "✅", "📨", empty to disable
    direct: true,     // send in DMs
    group: "mentions" // always|mentions|never
  }
}
```

### Media Handling

| Type | Limit |
|------|-------|
| Inbound | `mediaMaxMb` (default 50 MB) |
| Outbound | `agents.defaults.mediaMaxMb` (default 5 MB) |

- Audio sent as voice notes (PTT)
- Animated GIFs: send as MP4 with `gifPlayback: true`
- Caption only on first media item

### Multi-Account

```bash
moltbot channels login --account <id>
```

Credentials: `~/.clawdbot/credentials/whatsapp/<accountId>/creds.json`

### Troubleshooting

| Issue | Solution |
|-------|----------|
| Not linked | Run `moltbot channels login`, scan QR |
| Disconnected | Run `moltbot doctor` or relink |
| Bun issues | Use Node.js instead |

---

## Telegram

### Setup

1. Create bot via `@BotFather` → `/newbot`
2. Copy bot token
3. Configure in moltbot.json or env `TELEGRAM_BOT_TOKEN`

### Complete Configuration

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      allowFrom: ["123456789", "@username"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["987654321"],
      textChunkLimit: 4000,
      chunkMode: "length",
      mediaMaxMb: 5,
      historyLimit: 50,
      linkPreview: true,
      replyToMode: "first",     // off|first|all
      streamMode: "partial",    // off|partial|block
      reactionNotifications: "own",  // off|own|all
      reactionLevel: "minimal",      // off|ack|minimal|extensive
      groups: {
        "-1001234567890": { requireMention: false },
        "*": { requireMention: true }
      },
      capabilities: {
        inlineButtons: "allowlist"  // off|dm|group|all|allowlist
      },
      actions: {
        reactions: true,
        sendMessage: true,
        deleteMessage: true,
        sticker: false
      },
      customCommands: [
        { command: "backup", description: "Git backup" }
      ]
    }
  }
}
```

### Group Settings

```json5
{
  groups: {
    "-1001234567890": {
      requireMention: false,
      skills: ["skill1"],
      systemPrompt: "Extra instructions",
      topics: {
        "456": { requireMention: false }
      }
    },
    "*": { requireMention: true }
  }
}
```

**Privacy Mode:** Disable via `@BotFather` `/setprivacy` for full message visibility, or make bot admin.

### Inline Buttons

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "Choose:",
  buttons: [
    [
      {text: "Yes", callback_data: "yes"},
      {text: "No", callback_data: "no"}
    ]
  ]
}
```

### Stickers

- Static stickers (WEBP) process through vision
- Animated (TGS) and video (WEBM) skip processing
- Cache at `~/.clawdbot/telegram/sticker-cache.json`
- Enable sending: `actions.sticker: true`

### Action Gating (Telegram-specific)

Telegram supports disabling specific actions:

```json5
{
  actions: {
    reactions: true,
    sendMessage: true,   // Set to false to prevent sending messages
    deleteMessage: true
  }
}
```

**Note:** `actions.sendMessage: false` is Telegram-only. WhatsApp does NOT have this feature - it only has `actions.reactions`.

### Polls (NOT SUPPORTED)

**IMPORTANT:** Telegram polls are NOT supported via moltbot CLI. The `clawdbot message poll` command only works for Discord.

Error when attempting: `Action poll is not supported for provider telegram`

**Workarounds:**
1. **Numbered lists** (recommended) - Send items as numbered list, user replies with numbers. Agent can do this directly via `clawdbot message send`.
2. **Inline buttons** - Use buttons array (see above)
3. **Direct Telegram API** - Call `sendPoll` endpoint via `curl`

### Streaming (Draft Bubbles)

```json5
{
  streamMode: "partial",  // or "block"
  draftChunk: {
    minChars: 200,
    maxChars: 800,
    breakPreference: "paragraph"
  }
}
```

Requires Bot API 9.3+, private chats with topics enabled.

### Webhook Mode

```json5
{
  webhookUrl: "https://example.com/telegram",
  webhookSecret: "secret",
  webhookPath: "/telegram-webhook"
}
```

### Troubleshooting

| Issue | Solution |
|-------|----------|
| Bot silent in groups | Disable Privacy Mode, remove/re-add bot |
| IPv6 issues | Force IPv4 via `/etc/hosts` |
| Commands unavailable | Verify user authorized |

---

## Discord

### Setup

1. Create bot at discord.com/developers
2. Enable Message Content Intent + Server Members Intent
3. Copy bot token
4. Add bot to servers with message permissions

### Complete Configuration

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN",
      dm: {
        enabled: true,
        policy: "pairing",
        allowFrom: ["user_id"],
        groupEnabled: false
      },
      groupPolicy: "allowlist",  // open|allowlist|disabled
      textChunkLimit: 2000,
      maxLinesPerMessage: 17,
      mediaMaxMb: 8,
      historyLimit: 20,
      replyToMode: "off",       // off|first|all
      guilds: {
        "guild_id": {
          users: ["user_id"],
          requireMention: true,
          reactionNotifications: "own",
          channels: {
            "channel_id": {
              allow: true,
              requireMention: false,
              users: ["user_id"],
              skills: ["search"],
              systemPrompt: "Keep answers short."
            }
          }
        }
      },
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        messages: true,
        threads: true,
        pins: true,
        memberInfo: true,
        roleInfo: true,
        channelInfo: true,
        roles: false,        // disabled by default
        moderation: false    // disabled by default
      }
    }
  }
}
```

### Required Permissions

- View Channels, Send Messages, Read Message History
- Embed Links, Attach Files
- Add Reactions (optional), Use External Emojis (optional)

### Tool Actions

**Enabled by default:** reactions, stickers, polls, messages, threads, pins, memberInfo, roleInfo, channelInfo

**Disabled by default:** roles, moderation

### Troubleshooting

| Issue | Solution |
|-------|----------|
| "Used disallowed intents" | Enable Message Content + Server Members Intent |
| Bot connects but silent | Check intents, permissions, mention requirements |
| Name resolution fails | Use IDs or `<@id>` mentions instead of names |

---

## Slack

### Connection Modes

**Socket Mode (Default):**
- Requires App Token (`xapp-...`) + Bot Token (`xoxb-...`)
- No public webhook needed

**HTTP Mode:**
- Requires Bot Token + Signing Secret + webhook URL
- Set `mode: "http"`

### Complete Configuration

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",          // socket|http
      appToken: "xapp-...",
      botToken: "xoxb-...",
      userToken: "xoxp-...",   // optional, read-only by default
      userTokenReadOnly: true,
      dmPolicy: "pairing",
      groupPolicy: "allowlist",
      textChunkLimit: 4000,
      chunkMode: "length",
      mediaMaxMb: 20,
      replyToMode: "off",      // off|first|all
      replyToModeByChatType: {
        direct: "off",
        group: "off",
        channel: "first"
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true
      }
    }
  }
}
```

### Required Bot Token Scopes

`chat:write`, `im:write`, `channels:history`, `groups:history`, `im:history`, `mpim:history`, `channels:read`, `groups:read`, `im:read`, `mpim:read`, `users:read`, `reactions:read`, `reactions:write`, `pins:read`, `pins:write`, `emoji:read`, `files:write`

### Setup Steps

1. Create Slack app at api.slack.com/apps
2. Enable Socket Mode, generate App Token with `connections:write`
3. Install app, retrieve Bot Token
4. Subscribe to events: `message.*`, `app_mention`, `reaction_*`
5. Invite bot to channels
6. Enable Messages Tab in App Home for DMs

### Reply Threading

```json5
{
  replyToMode: "first",  // Initial reply threads
  replyToModeByChatType: {
    direct: "all",       // Thread all DM replies
    channel: "first"     // First reply threads in channels
  }
}
```

### HTTP Mode Config

```json5
{
  mode: "http",
  botToken: "xoxb-...",
  signingSecret: "your-signing-secret",
  webhookPath: "/slack/events"
}
```

---

## Signal

### Setup

1. Get separate Signal number for bot
2. Install `signal-cli` (requires Java)
3. Link device: `signal-cli link -n "Moltbot"`

### Configuration

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "/usr/local/bin/signal-cli",
      dmPolicy: "pairing",
      groupPolicy: "allowlist",
      textChunkLimit: 4000,
      mediaMaxMb: 8,
      reactionNotifications: "own"
    }
  }
}
```

---

## Common Patterns

### Multi-Channel Same Agent

```json5
{
  channels: {
    whatsapp: { enabled: true, dmPolicy: "pairing" },
    telegram: { enabled: true, botToken: "..." },
    discord: { enabled: true, token: "..." }
  }
}
```

### Allowlist Multiple Users

```json5
{
  dmPolicy: "allowlist",
  allowFrom: ["+15551234567", "+15559876543"]
}
```

### Open Access with Mention Required

```json5
{
  dmPolicy: "open",
  allowFrom: ["*"],
  groups: { "*": { requireMention: true } }
}
```

---

## Continuous Improvement

**Always verify with current docs:** Before implementing, fetch the relevant page from https://docs.molt.bot/channels/<channel> to check for updates.

When using Clawdbot and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/clawdbot-guide/references/channels-messaging.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage
