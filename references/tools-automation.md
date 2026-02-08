# Automation Tools

## Cron Jobs

Cron is the Gateway's built-in scheduler that persists jobs, wakes agents at scheduled times, and can deliver output to chat.

### Overview

- Runs inside Gateway (not the model)
- Stores jobs at `~/.openclaw/cron/`
- Jobs use stable `jobId` identifiers
- Run history at `~/.openclaw/cron/runs/<jobId>.jsonl`

### Schedule Types

| Type | Description | Example |
|------|-------------|---------|
| `at` | One-shot timestamp (ms since epoch or ISO 8601) | `"2026-01-12T18:00:00Z"` or `"20m"` |
| `every` | Fixed interval in milliseconds | `3600000` (1 hour) |
| `cron` | 5-field cron expression | `"0 7 * * *"` (7 AM daily) |

For cron expressions, omitted timezone defaults to Gateway host's local timezone.

### Execution Modes

#### Main Session (System Events)

```json5
{
  payload: {
    kind: "systemEvent",
    text: "Check calendar",
    wakeMode: "now"  // or "next-heartbeat" (default)
  }
}
```

- Enqueues event for heartbeat processing
- `"now"` triggers immediate heartbeat
- `"next-heartbeat"` waits for scheduled heartbeat

#### Isolated Session (Dedicated Turns)

```json5
{
  payload: {
    kind: "agentTurn",
    message: "Summarize inbox",
    model: "anthropic/claude-sonnet-4",
    thinking: "medium",
    timeoutSeconds: 300,
    deliver: true,
    channel: "whatsapp",
    to: "+15551234567"
  }
}
```

- Fresh session ID each run (no conversation carry-over)
- Session key: `cron:<jobId>`
- Summary posted to main session

**Isolation options:**
- `postToMainPrefix` — Prefix for main-session system event
- `postToMainMode` — `"summary"` (default) or `"full"`
- `postToMainMaxChars` — Max chars when mode is `"full"` (default: 8000)

### Model & Thinking Overrides

For isolated jobs only:

| Parameter | Description |
|-----------|-------------|
| `model` | Provider/model string or alias (`"opus"`, `"sonnet"`) |
| `thinking` | Level: `off`, `minimal`, `low`, `medium`, `high`, `xhigh` |

Resolution priority: Job payload → Hook defaults → Agent config default

### Delivery Configuration

**Channel targeting:**
- If `to` is set, output auto-delivers even without `deliver` flag
- Use `deliver: true` for last-route delivery without explicit `to`
- Use `deliver: false` to keep output internal despite `to` presence

**Target formats:**
- WhatsApp: `+15551234567`
- Telegram: `-1001234567890` or `-1001234567890:topic:123`
- Discord/Slack/Mattermost: `channel:<id>` or `user:<id>`

### CLI Commands

**Add one-shot reminder (UTC):**
```bash
openclaw cron add --name "Send reminder" \
  --at "2026-01-12T18:00:00Z" \
  --session main \
  --system-event "Reminder text" \
  --wake now \
  --delete-after-run
```

**Add relative reminder (20 minutes):**
```bash
openclaw cron add --name "Calendar check" \
  --at "20m" \
  --session main \
  --system-event "Check calendar." \
  --wake now
```

**Recurring isolated job with WhatsApp delivery:**
```bash
openclaw cron add --name "Morning status" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize inbox + calendar." \
  --deliver \
  --channel whatsapp \
  --to "+15551234567"
```

**With model & thinking overrides:**
```bash
openclaw cron add --name "Deep analysis" \
  --cron "0 6 * * 1" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Weekly analysis." \
  --model "opus" \
  --thinking high \
  --deliver \
  --channel whatsapp \
  --to "+15551234567"
```

**Agent-specific jobs:**
```bash
openclaw cron add --name "Ops sweep" \
  --cron "0 6 * * *" \
  --session isolated \
  --message "Check ops queue" \
  --agent ops
```

**Management commands:**
```bash
openclaw cron list                    # List all jobs
openclaw cron show <jobId>            # Job details
openclaw cron edit <jobId> --message "Updated" --model opus
openclaw cron run <jobId> --force     # Manual run
openclaw cron runs --id <jobId> --limit 50  # View history
openclaw cron remove <jobId>          # Delete job
```

### Configuration

```json5
{
  cron: {
    enabled: true,
    store: "~/.openclaw/cron/jobs.json",
    maxConcurrentRuns: 1
  }
}
```

Disable: `cron.enabled: false` or `OPENCLAW_SKIP_CRON=1`

### Troubleshooting

| Issue | Solution |
|-------|----------|
| Nothing runs | Verify `cron.enabled: true`, no `OPENCLAW_SKIP_CRON=1`, Gateway running |
| Wrong delivery | Use explicit format like `-100…:topic:<id>` for Telegram topics |

## Webhooks

HTTP endpoints for external triggers.

### Configuration

```json5
{
  webhooks: {
    enabled: true,
    endpoints: [
      {
        id: "github-push",
        path: "/webhook/github",
        secret: "your-webhook-secret",
        channel: "slack",
        to: "#dev-alerts"
      }
    ]
  }
}
```

### Session Handling

Webhooks create sessions with key: `hook:<uuid>`

### Security

- Configure shared secrets for signature validation
- Validate signatures for known providers (GitHub, etc.)

## Polling

Periodic checks for external events.

### Configuration

```json5
{
  polling: {
    sources: [
      {
        id: "email-check",
        interval: 300,
        command: "check-emails.sh",
        channel: "telegram",
        to: "123456789"
      }
    ]
  }
}
```

### Polling vs Cron

| Feature | Polling | Cron |
|---------|---------|------|
| Interval | Fixed duration | Crontab schedule |
| Use case | Continuous monitoring | Specific times |
| Session | Per-check | Per-job |

## Gmail Pub/Sub Integration

Real-time Gmail notifications via Google Cloud Pub/Sub.

### Configuration

```json5
{
  integrations: {
    gmail: {
      enabled: true,
      pubsub: {
        projectId: "your-project",
        topicName: "gmail-notifications",
        subscriptionName: "openclaw-gmail"
      }
    }
  }
}
```

## Gateway API Methods

| Method | Description |
|--------|-------------|
| `cron.list` | List all jobs |
| `cron.status` | Get job status |
| `cron.add` | Create new job |
| `cron.update` | Modify job |
| `cron.remove` | Delete job |
| `cron.run` | Force or run due jobs |
| `cron.runs` | Retrieve run history |

## Best Practices

### Cron Jobs

1. Use descriptive job names
2. Set `deleteAfterRun: true` for one-shot reminders
3. Use timezone-aware cron expressions
4. Configure appropriate delivery channels
5. Use isolated sessions for long-running tasks

### Webhooks

1. Always validate webhook signatures
2. Use HTTPS endpoints
3. Configure rate limiting
4. Log webhook activity for debugging

### Polling

1. Set reasonable intervals (don't poll too frequently)
2. Implement backoff on failures
3. Use webhooks instead when available

---

## Continuous Improvement

**Always verify with current docs:** Before implementing, fetch the relevant page from https://docs.molt.bot/automation/cron-jobs to check for updates.

When using OpenClaw and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/openclaw-guide/references/tools-automation.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage
