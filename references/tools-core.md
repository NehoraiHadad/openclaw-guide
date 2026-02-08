# Core Execution Tools

## Exec Tool

The exec tool executes shell commands in a workspace, supporting both synchronous foreground and asynchronous background execution.

### Complete Parameters

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| `command` | required | — | Shell command to execute |
| `workdir` | string | cwd | Working directory for execution |
| `env` | object | — | Environment variable key/value overrides |
| `yieldMs` | integer | 10000 | Delay before auto-backgrounding |
| `background` | boolean | false | Immediately background the process |
| `timeout` | seconds | 1800 | Kill command on expiry |
| `pty` | boolean | — | Run in pseudo-terminal (TTY-only CLIs, agents, UIs) |
| `host` | enum | sandbox | Execution location: `sandbox`, `gateway`, or `node` |
| `security` | enum | varies | Enforcement mode: `deny`, `allowlist`, or `full` |
| `ask` | enum | on-miss | Approval prompts: `off`, `on-miss`, or `always` |
| `node` | string | — | Node ID/name when `host=node` |
| `elevated` | boolean | — | Request elevated mode (gateway host only) |

### Execution Hosts

#### Sandbox (Default)

Container-based isolation; runs `sh -lc` login shell internally. No approvals needed if sandboxing is off.

**Important:** Sandboxing is **off by default**. If sandboxing is off, `host=sandbox` runs directly on the gateway host (no container) and **does not require approvals**.

#### Gateway

Host machine execution with approval controls via `~/.openclaw/exec-approvals.json`. Merges your login-shell `PATH` into the exec environment unless the call already sets `env.PATH`.

#### Node

Requires a paired companion app or headless node host. Only receives explicit env overrides; `tools.exec.pathPrepend` applies only if the call sets `env.PATH`.

### Security Modes

| Mode | Description |
|------|-------------|
| `deny` | Blocks execution (default for sandbox) |
| `allowlist` | Permits only allowlisted binaries; rejects chaining (`;`, `&&`, `\|\|`) and redirections |
| `full` | Unrestricted execution; forced when `elevated=true` with `security=full` |

**Allowlist enforcement matches resolved binary paths only** (no basename matches).

### Approval System

When approvals are required, the exec tool returns immediately with `status: "approval-pending"` and an approval ID. Subsequent system events (`Exec finished`, `Exec denied`, `Exec running`) notify completion or timeout.

Configuration: `tools.exec.approvalRunningNoticeMs` (default 10000ms) controls when a "running" notice emits.

### PATH Handling

| Host | Behavior |
|------|----------|
| Gateway | Prepends login-shell PATH; daemon PATH is minimal (`/opt/homebrew/bin`, `/usr/local/bin`, `/usr/bin`, `/bin`) |
| Sandbox | Sources `/etc/profile`, then prepends overrides via internal env var |
| Node | Only respects explicit overrides; macOS nodes drop PATH overrides entirely |

Use `tools.exec.pathPrepend` configuration to inject custom directories.

### Session Overrides (`/exec`)

Set per-session defaults without modifying config:

```
/exec host=gateway security=allowlist ask=on-miss node=mac-1
```

Only authorized senders (channel allowlists, pairing, `commands.useAccessGroups`) may use this command.

### Configuration Example

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
      notifyOnExit: true,
      approvalRunningNoticeMs: 10000,
      host: "sandbox",
      security: "deny",
      ask: "on-miss"
    }
  }
}
```

### Safe Binaries

Configuration option `tools.exec.safeBins` specifies stdin-only binaries that bypass allowlist requirements, streamlining allowlist-mode workflows.

## Background Process Management (`/process`)

Background processes are scoped per agent; `/process` only sees sessions from the same agent.

### Actions

| Action | Description |
|--------|-------------|
| `poll` | Check process status and output |
| `send-keys` | Send keyboard input (e.g., `["C-c"]` for Ctrl+C) |
| `submit` | Submit command to process stdin |
| `paste` | Paste text with newlines to process |

### Examples

**Background with polling:**
```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

**Interactive control (tmux-style):**
```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"submit","sessionId":"<id>"}
{"tool":"process","action":"paste","sessionId":"<id>","text":"line1\nline2\n"}
```

## Elevated Mode

Elevated mode allows execution on gateway host instead of sandbox.

### States

| Command | Effect |
|---------|--------|
| `/elevated on\|ask` | Run on host, preserve approvals |
| `/elevated full` | Run on host, auto-approve |
| `/elevated off` | Disable elevated mode |

### Authorization Gates (All Must Pass)

1. Global: `tools.elevated.enabled` must be true
2. Sender: `tools.elevated.allowFrom` validates sender
3. Per-agent: `agents.list[].tools.elevated` can restrict
4. Channel fallback: Uses channel allowFrom if no explicit list

**Important:** Elevated is ignored when sandboxing is off (exec already runs on the host).

### Configuration

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: ["+15551234567", "discord:123456789"]
    }
  }
}
```

## Apply_Patch Subtool (Experimental)

Structured multi-file edits for OpenAI/Codex models. Enable explicitly:

```json5
{
  tools: {
    exec: {
      applyPatch: { enabled: true, allowModels: ["gpt-5.2"] }
    }
  }
}
```

Tool policy applies; `allow: ["exec"]` implicitly permits `apply_patch`.

## Exec in Bot-Orchestrated Workflows

The exec tool is a key component of **bot-orchestrated workflow skills** - where a skill teaches the bot to combine exec output with native tools like `message` or `poll`.

**Pattern:** exec → parse output → act with native tools

**Example:** A skill runs `node filter-groups.mjs --json`, parses the JSON array, then sends polls to the user via the `message` tool.

See `references/skills-system.md` → "Bot-Orchestrated Workflow Skills" for the full pattern and best practices.

---

## Critical Gotchas

1. **Sandboxing off by default** - `host=sandbox` runs on gateway host without approvals if sandboxing disabled
2. **Shell preference** - On non-Windows, exec uses `SHELL` env var; if fish, prefers bash/sh from PATH
3. **Elevated ignored when unsandboxed** - Already running on host
4. **Node binding** - Per-agent node assignment: `agents.list[0].tools.exec.node`
5. **Allowlist matches resolved paths only** - No basename matches

---

## Continuous Improvement

**Always verify with current docs:** Before implementing, fetch the relevant page from https://docs.molt.bot/tools/exec to check for updates.

When using OpenClaw and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/openclaw-guide/references/tools-core.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage
