# Direct Baileys Access (WhatsApp)

Moltbot uses the `@whiskeysockets/baileys` library for WhatsApp Web connections. In some cases, you may need to access Baileys directly to perform operations not exposed through Moltbot's API.

## Scripts Location

Custom Baileys scripts should be placed in the workspace scripts folder:

```
<workspace>/scripts/whatsapp/
├── list-groups.mjs      # List all WhatsApp groups
├── filter-groups.mjs    # Filter groups by keywords
└── passive-monitor.mjs  # Monitor messages passively
```

**Standard location:** `~/clawd/scripts/whatsapp/` (adjust based on your `agents.defaults.workspace` setting)

The agent can run these scripts via the exec tool. See `TOOLS.md` in the workspace for usage documentation.

## When to Use Direct Baileys Access

- **List all groups** - Moltbot's `directory groups list` only reads from config, not live data
- **Passive monitoring** - No built-in way to receive messages without agent responding
- **Bulk operations** - Get metadata for all groups/contacts at once
- **Features not exposed** - Some Baileys functions aren't available in Moltbot

## Important Limitations

**WhatsApp allows only ONE active connection per account.** Running a Baileys script will:
1. Disconnect Moltbot temporarily
2. Moltbot auto-reconnects after the script exits
3. Brief interruption in service (a few seconds)

## Credentials Location

Baileys auth files are stored at:
```
~/.clawdbot/credentials/whatsapp/<accountId>/
```

Files include:
- `creds.json` - Main credentials
- `app-state-sync-key-*.json` - Sync keys
- `lid-mapping-*.json` - Contact mappings

## List All Groups Script

Location: `~/clawd/scripts/whatsapp/list-groups.mjs`

```javascript
#!/usr/bin/env node
/**
 * List all WhatsApp groups from Baileys connection
 *
 * WARNING: This will briefly disconnect Moltbot. It will auto-reconnect after.
 *
 * Usage: node list-whatsapp-groups.mjs [accountId]
 * Example: node list-whatsapp-groups.mjs personal
 */

// Use Baileys from Moltbot's installation
const CLAWDBOT_PATH = '/home/nehor/.nvm/versions/node/v24.13.0/lib/node_modules/clawdbot/node_modules';
const { makeWASocket, useMultiFileAuthState, fetchLatestBaileysVersion, makeCacheableSignalKeyStore } = await import(`${CLAWDBOT_PATH}/@whiskeysockets/baileys/lib/index.js`);
const pino = (await import(`${CLAWDBOT_PATH}/pino/pino.js`)).default;

import path from 'path';
import os from 'os';

const accountId = process.argv[2] || 'personal';
const authDir = path.join(os.homedir(), '.clawdbot', 'credentials', 'whatsapp', accountId);

console.log(`Using auth from: ${authDir}`);
console.log(`Account: ${accountId}`);
console.log('');
console.log('WARNING: This will briefly disconnect Moltbot from WhatsApp.');
console.log('Moltbot will auto-reconnect after this script exits.');
console.log('');

const logger = pino({ level: 'silent' });

async function main() {
  const { state, saveCreds } = await useMultiFileAuthState(authDir);
  const { version } = await fetchLatestBaileysVersion();

  const sock = makeWASocket({
    auth: {
      creds: state.creds,
      keys: makeCacheableSignalKeyStore(state.keys, logger),
    },
    version,
    logger,
    printQRInTerminal: false,
    syncFullHistory: false,
    markOnlineOnConnect: false,
  });

  sock.ev.on('creds.update', saveCreds);

  // Wait for connection
  await new Promise((resolve, reject) => {
    const timeout = setTimeout(() => reject(new Error('Connection timeout')), 30000);

    sock.ev.on('connection.update', (update) => {
      if (update.connection === 'open') {
        clearTimeout(timeout);
        resolve();
      }
      if (update.connection === 'close') {
        clearTimeout(timeout);
        reject(new Error('Connection closed'));
      }
    });
  });

  console.log('Connected! Fetching groups...\n');

  // Fetch all groups
  const groups = await sock.groupFetchAllParticipating();

  console.log(`Found ${Object.keys(groups).length} groups:\n`);
  console.log('=' .repeat(80));

  for (const [jid, meta] of Object.entries(groups)) {
    console.log(`ID: ${jid}`);
    console.log(`Name: ${meta.subject}`);
    console.log(`Participants: ${meta.participants?.length || 0}`);
    console.log('-'.repeat(80));
  }

  // Output as JSON for easy parsing
  const outputPath = path.join(os.homedir(), 'whatsapp-groups.json');
  const fs = await import('fs');
  fs.writeFileSync(outputPath, JSON.stringify(groups, null, 2));
  console.log(`\nFull data saved to: ${outputPath}`);

  // Disconnect gracefully
  sock.end();
  process.exit(0);
}

main().catch((err) => {
  console.error('Error:', err.message);
  process.exit(1);
});
```

### Usage

```bash
# List all groups from personal account
node ~/clawd/scripts/whatsapp/list-groups.mjs personal

# List from agent account
node ~/clawd/scripts/whatsapp/list-groups.mjs agent

# Output saved to ~/whatsapp-groups.json
```

## Filter Groups Script

Location: `~/clawd/scripts/whatsapp/filter-groups.mjs`

A more powerful script with filtering, exclusion, and caching capabilities.

```javascript
#!/usr/bin/env node
/**
 * List and filter WhatsApp groups from Baileys connection
 *
 * Usage:
 *   node whatsapp-groups-filter.mjs [accountId] [--filter keyword1,keyword2,...]
 *   node whatsapp-groups-filter.mjs personal --filter גן,בית ספר,עבודה
 *   node whatsapp-groups-filter.mjs personal --filter school,work,family
 *   node whatsapp-groups-filter.mjs personal --exclude spam,מכירות
 *   node whatsapp-groups-filter.mjs personal --min-participants 10
 *   node whatsapp-groups-filter.mjs personal --json
 *   node whatsapp-groups-filter.mjs personal --cached  (use cached data, no reconnect)
 */

import path from 'path';
import os from 'os';
import fs from 'fs';

// Parse arguments
const args = process.argv.slice(2);
let accountId = 'personal';
let filters = [];
let excludes = [];
let minParticipants = 0;
let jsonOutput = false;
let useCached = false;

for (let i = 0; i < args.length; i++) {
  if (args[i] === '--filter' && args[i + 1]) {
    filters = args[i + 1].split(',').map(f => f.trim().toLowerCase());
    i++;
  } else if (args[i] === '--exclude' && args[i + 1]) {
    excludes = args[i + 1].split(',').map(f => f.trim().toLowerCase());
    i++;
  } else if (args[i] === '--min-participants' && args[i + 1]) {
    minParticipants = parseInt(args[i + 1], 10);
    i++;
  } else if (args[i] === '--json') {
    jsonOutput = true;
  } else if (args[i] === '--cached') {
    useCached = true;
  } else if (!args[i].startsWith('--')) {
    accountId = args[i];
  }
}

const cacheFile = path.join(os.homedir(), 'whatsapp-groups.json');

function matchesFilters(groupName, filters) {
  if (!filters.length) return true;
  const nameLower = (groupName || '').toLowerCase();
  return filters.some(f => nameLower.includes(f));
}

function matchesExcludes(groupName, excludes) {
  if (!excludes.length) return false;
  const nameLower = (groupName || '').toLowerCase();
  return excludes.some(f => nameLower.includes(f));
}

async function fetchGroups() {
  const CLAWDBOT_PATH = '/home/nehor/.nvm/versions/node/v24.13.0/lib/node_modules/clawdbot/node_modules';
  const { makeWASocket, useMultiFileAuthState, fetchLatestBaileysVersion, makeCacheableSignalKeyStore } = await import(`${CLAWDBOT_PATH}/@whiskeysockets/baileys/lib/index.js`);
  const pino = (await import(`${CLAWDBOT_PATH}/pino/pino.js`)).default;

  const authDir = path.join(os.homedir(), '.clawdbot', 'credentials', 'whatsapp', accountId);
  if (!jsonOutput) console.log(`Connecting to WhatsApp (account: ${accountId})...\n`);

  const logger = pino({ level: 'silent' });
  const { state, saveCreds } = await useMultiFileAuthState(authDir);
  const { version } = await fetchLatestBaileysVersion();

  const sock = makeWASocket({
    auth: { creds: state.creds, keys: makeCacheableSignalKeyStore(state.keys, logger) },
    version, logger, printQRInTerminal: false, syncFullHistory: false, markOnlineOnConnect: false,
  });

  sock.ev.on('creds.update', saveCreds);

  await new Promise((resolve, reject) => {
    const timeout = setTimeout(() => reject(new Error('Connection timeout')), 30000);
    sock.ev.on('connection.update', (update) => {
      if (update.connection === 'open') { clearTimeout(timeout); resolve(); }
      if (update.connection === 'close') { clearTimeout(timeout); reject(new Error('Connection closed')); }
    });
  });

  const groups = await sock.groupFetchAllParticipating();
  fs.writeFileSync(cacheFile, JSON.stringify(groups, null, 2));
  sock.end();
  return groups;
}

async function main() {
  let groups = useCached
    ? JSON.parse(fs.readFileSync(cacheFile, 'utf-8'))
    : await fetchGroups();

  const filtered = Object.entries(groups)
    .map(([jid, meta]) => ({ jid, name: meta.subject || 'undefined', participants: meta.participants?.length || 0 }))
    .filter(g => matchesFilters(g.name, filters))
    .filter(g => !matchesExcludes(g.name, excludes))
    .filter(g => g.participants >= minParticipants)
    .sort((a, b) => b.participants - a.participants);

  if (jsonOutput) {
    console.log(JSON.stringify(filtered, null, 2));
  } else {
    console.log(`Found ${filtered.length} groups:\n`);
    for (const g of filtered) {
      console.log(`${g.name} (${g.participants} members) - ${g.jid}`);
    }
  }
}

main().catch(console.error);
```

### Filter Options

| Option | Description | Example |
|--------|-------------|---------|
| `--filter` | Include groups matching keywords | `--filter גן,בית ספר` |
| `--exclude` | Exclude groups matching keywords | `--exclude spam,מכירות` |
| `--min-participants` | Minimum group size | `--min-participants 50` |
| `--json` | JSON output for programmatic use | `--json` |
| `--cached` | Use cached data (no reconnect) | `--cached` |

### Common Filter Examples

```bash
# Kindergarten groups
node ~/clawd/scripts/whatsapp/filter-groups.mjs personal --cached --filter גן,גנון,פעוטון

# School groups
node ~/clawd/scripts/whatsapp/filter-groups.mjs personal --cached --filter "בית ספר,כיתה,הורים"

# Work groups
node ~/clawd/scripts/whatsapp/filter-groups.mjs personal --cached --filter עבודה,work,job,משרד,צוות

# Family groups
node ~/clawd/scripts/whatsapp/filter-groups.mjs personal --cached --filter משפחה,family

# Large active groups
node ~/clawd/scripts/whatsapp/filter-groups.mjs personal --cached --min-participants 100

# Exclude spam, get JSON for processing
node ~/clawd/scripts/whatsapp/filter-groups.mjs personal --cached --exclude מכירות,פרסום --json
```

## Available Baileys Functions

Once connected, the `sock` object provides these methods:

### Group Operations

| Function | Description |
|----------|-------------|
| `sock.groupFetchAllParticipating()` | List all groups account is in |
| `sock.groupMetadata(jid)` | Get group details, participants, admins |
| `sock.groupCreate(name, participants)` | Create a new group |
| `sock.groupLeave(jid)` | Leave a group |
| `sock.groupUpdateSubject(jid, name)` | Change group name |
| `sock.groupUpdateDescription(jid, desc)` | Change group description |
| `sock.groupParticipantsUpdate(jid, participants, action)` | Add/remove/promote/demote members |
| `sock.groupInviteCode(jid)` | Get group invite link |
| `sock.groupRevokeInvite(jid)` | Revoke invite link |

### Profile Operations

| Function | Description |
|----------|-------------|
| `sock.profilePictureUrl(jid, 'image')` | Get profile/group picture URL |
| `sock.updateProfilePicture(jid, image)` | Update profile picture |
| `sock.fetchStatus(jid)` | Get contact's status/about |
| `sock.updateProfileStatus(status)` | Update your status |

### Contact Operations

| Function | Description |
|----------|-------------|
| `sock.onWhatsApp(phone)` | Check if number is on WhatsApp |
| `sock.getBusinessProfile(jid)` | Get business profile info |
| `sock.fetchBlocklist()` | Get blocked contacts |
| `sock.updateBlockStatus(jid, action)` | Block/unblock contact |

### Message Operations

| Function | Description |
|----------|-------------|
| `sock.sendMessage(jid, content)` | Send a message |
| `sock.readMessages([key])` | Mark messages as read |
| `sock.sendPresenceUpdate(presence, jid)` | Send typing/online status |

### Chat Operations

| Function | Description |
|----------|-------------|
| `sock.chatModify({ archive: true }, jid)` | Archive chat |
| `sock.chatModify({ mute: duration }, jid)` | Mute chat |
| `sock.chatModify({ pin: true }, jid)` | Pin chat |
| `sock.chatModify({ delete: true }, jid)` | Delete chat |

### Event Listeners

| Event | Description |
|-------|-------------|
| `sock.ev.on('messages.upsert', ...)` | **Real-time incoming messages** |
| `sock.ev.on('messages.update', ...)` | Message status updates |
| `sock.ev.on('connection.update', ...)` | Connection state changes |
| `sock.ev.on('creds.update', ...)` | Credentials updated |
| `sock.ev.on('groups.upsert', ...)` | New groups |
| `sock.ev.on('groups.update', ...)` | Group info changes |
| `sock.ev.on('group-participants.update', ...)` | Member changes |
| `sock.ev.on('presence.update', ...)` | Contact online status |

## Passive Message Monitor Script

For true passive monitoring (receives all messages, logs to file, no agent wake-up):

```javascript
#!/usr/bin/env node
/**
 * Passive WhatsApp message monitor
 * Logs all messages to file without responding
 *
 * WARNING: Replaces Moltbot connection. Run one or the other.
 *
 * Usage: node whatsapp-passive-monitor.mjs [accountId]
 */

const CLAWDBOT_PATH = '/home/nehor/.nvm/versions/node/v24.13.0/lib/node_modules/clawdbot/node_modules';
const { makeWASocket, useMultiFileAuthState, fetchLatestBaileysVersion, makeCacheableSignalKeyStore } = await import(`${CLAWDBOT_PATH}/@whiskeysockets/baileys/lib/index.js`);
const pino = (await import(`${CLAWDBOT_PATH}/pino/pino.js`)).default;

import path from 'path';
import os from 'os';
import fs from 'fs';

const accountId = process.argv[2] || 'personal';
const authDir = path.join(os.homedir(), '.clawdbot', 'credentials', 'whatsapp', accountId);
const logFile = path.join(os.homedir(), `whatsapp-messages-${accountId}.jsonl`);

console.log(`Passive monitor for account: ${accountId}`);
console.log(`Logging to: ${logFile}`);
console.log('Press Ctrl+C to stop\n');

const logger = pino({ level: 'silent' });

async function main() {
  const { state, saveCreds } = await useMultiFileAuthState(authDir);
  const { version } = await fetchLatestBaileysVersion();

  const sock = makeWASocket({
    auth: {
      creds: state.creds,
      keys: makeCacheableSignalKeyStore(state.keys, logger),
    },
    version,
    logger,
    printQRInTerminal: false,
    syncFullHistory: false,
    markOnlineOnConnect: false,
  });

  sock.ev.on('creds.update', saveCreds);

  sock.ev.on('connection.update', (update) => {
    if (update.connection === 'open') {
      console.log('Connected! Monitoring messages...\n');
    }
    if (update.connection === 'close') {
      console.log('Disconnected. Exiting.');
      process.exit(0);
    }
  });

  // Monitor ALL incoming messages
  sock.ev.on('messages.upsert', (upsert) => {
    if (upsert.type !== 'notify') return;

    for (const msg of upsert.messages) {
      const entry = {
        timestamp: new Date().toISOString(),
        from: msg.key.remoteJid,
        fromMe: msg.key.fromMe,
        participant: msg.key.participant, // sender in group
        pushName: msg.pushName,
        messageType: Object.keys(msg.message || {})[0],
        text: msg.message?.conversation ||
              msg.message?.extendedTextMessage?.text ||
              msg.message?.imageMessage?.caption ||
              msg.message?.videoMessage?.caption ||
              '[non-text]',
      };

      // Log to console
      const isGroup = entry.from?.endsWith('@g.us');
      console.log(`[${entry.timestamp}] ${isGroup ? 'GROUP' : 'DM'} ${entry.from}`);
      console.log(`  From: ${entry.pushName || entry.participant || 'unknown'}`);
      console.log(`  Text: ${entry.text.substring(0, 100)}${entry.text.length > 100 ? '...' : ''}`);
      console.log('');

      // Append to log file
      fs.appendFileSync(logFile, JSON.stringify(entry) + '\n');
    }
  });

  // Handle graceful shutdown
  process.on('SIGINT', () => {
    console.log('\nShutting down...');
    sock.end();
    process.exit(0);
  });
}

main().catch((err) => {
  console.error('Error:', err.message);
  process.exit(1);
});
```

### Usage

```bash
# Start passive monitor (replaces Moltbot connection)
node ~/clawd/scripts/whatsapp/passive-monitor.mjs personal

# Messages logged to ~/whatsapp-messages-personal.jsonl

# When done, restart Moltbot
clawdbot gateway restart
```

## Integration with Moltbot Agent (Future)

These scripts can potentially be converted to agent tools:
1. Create an MCP server that exposes Baileys functions
2. Or use the exec tool to run scripts and parse output
3. Or request Moltbot to add native support for these operations

## Known Limitations

### Activity Timestamps (Last Message Time)

**Problem:** You cannot query "when was the last message in this group" on-demand.

**Why:** The `conversationTimestamp` field (which tracks last message time) is only delivered during **initial QR pairing** via the `messaging-history.set` event. After that:
- `fetchMessageHistory()` requires an existing message key as anchor
- `chats.update` events only fire on new activity
- There's no API to query historical timestamps

**Workarounds:**

| Approach | Trade-off |
|----------|-----------|
| Re-pair (scan QR) | One-time sync gets all timestamps |
| Build local DB | Listen to `messages.upsert`, track over time |
| Use metadata heuristics | `subjectTime`/`descTime` show admin activity, not messages |
| Manual curation | Filter by keyword, manually identify active groups |

**Reference:** The [whatsapp-mcp-ts](https://github.com/jlucaso1/whatsapp-mcp-ts) project builds a SQLite database by listening to message events, enabling activity-based sorting.

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "Connection closed" immediately | Moltbot may have reconnected; stop Moltbot first |
| "Auth file not found" | Check `accountId` matches existing account |
| Script hangs | Connection timeout; check network/WhatsApp status |
| QR code requested | Account not linked; run `clawdbot channels login` first |

## Continuous Improvement

When discovering new Baileys capabilities or patterns:
1. Update this file at `~/.claude/skills/clawdbot-guide/references/baileys-direct-access.md`
2. Add new script examples for common operations
3. Document any gotchas or edge cases discovered
