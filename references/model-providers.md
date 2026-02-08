# Model Providers Reference

## Overview

OpenClaw supports 19+ LLM providers through a unified `provider/model` naming convention. Providers fall into three categories: **built-in** (Pi-AI catalog, zero config beyond auth), **custom/proxy** (OpenAI/Anthropic-compatible endpoints you configure), and **local** (on-device inference via Ollama, LM Studio, etc.).

Model IDs always use the format `provider/model-name`. For multi-slash model IDs (OpenRouter-style), include the full provider prefix: `openrouter/anthropic/claude-sonnet-4-5`.

---

## Built-in Providers (Pi-AI Catalog)

These providers require only authentication setup -- no additional `models.providers` configuration needed.

| Provider | Provider ID | Auth Method | Env Variable | Example Model ID |
|----------|-------------|-------------|--------------|------------------|
| Anthropic | `anthropic` | API key or setup-token | `ANTHROPIC_API_KEY` | `anthropic/claude-opus-4-6` |
| OpenAI | `openai` | API key | `OPENAI_API_KEY` | `openai/gpt-5.1-codex` |
| OpenAI Codex | `openai-codex` | OAuth (ChatGPT login) | -- | `openai-codex/gpt-5.3-codex` |
| OpenCode Zen | `opencode` | API key | `OPENCODE_API_KEY` | `opencode/claude-opus-4-6` |
| Google Gemini | `google` | API key | `GEMINI_API_KEY` | `google/gemini-3-pro-preview` |
| Google Vertex | `google-vertex` | Specific auth flow | -- | -- |
| Google Antigravity | `google-antigravity` | OAuth (Google account) | -- | -- |
| Google Gemini CLI | `google-gemini-cli` | CLI auth flow | -- | `google-gemini-cli/gemini-3-flash-preview` |
| OpenRouter | `openrouter` | API key | `OPENROUTER_API_KEY` | `openrouter/anthropic/claude-sonnet-4-5` |
| Vercel AI Gateway | `vercel-ai-gateway` | API key | `AI_GATEWAY_API_KEY` | `vercel-ai-gateway/anthropic/claude-opus-4.6` |
| Z.AI (GLM) | `zai` | API key | `ZAI_API_KEY` | `zai/glm-4.7` |
| Amazon Bedrock | `amazon-bedrock` | AWS credentials | `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` | `amazon-bedrock/us.anthropic.claude-opus-4-6-v1:0` |
| xAI | `xai` | API key | `XAI_API_KEY` | Custom models |
| Groq | `groq` | API key | `GROQ_API_KEY` | Custom models |
| Cerebras | `cerebras` | API key | `CEREBRAS_API_KEY` | Custom models |
| Mistral | `mistral` | API key | `MISTRAL_API_KEY` | Custom models |
| GitHub Copilot | `github-copilot` | Token | `COPILOT_GITHUB_TOKEN` | Custom models |
| Venice AI | `venice` | API key | -- | `venice/llama-3.3-70b` |

---

## Provider-Specific Details

### Anthropic

**Auth Options:**

| Method | Setup Command | Best For |
|--------|---------------|----------|
| API Key | `openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"` | Usage-based billing |
| Setup-Token | `openclaw models auth setup-token --provider anthropic` | Claude subscription users |
| Paste Token | `openclaw models auth paste-token --provider anthropic` | Pre-generated tokens |

**Generate a setup-token:** Run `claude setup-token` in the Claude Code CLI, then paste into OpenClaw.

**Prompt Caching (API Key only):**

| `cacheRetention` | Duration | Notes |
|-------------------|----------|-------|
| `none` | Disabled | No caching |
| `short` | 5 minutes | Default for API key auth |
| `long` | 1 hour | Requires beta flag (auto-included) |

The legacy parameter `cacheControlTtl` is still supported (`"5m"` maps to `short`, `"1h"` maps to `long`).

**Quirks:**
- Subscription auth does NOT support prompt caching
- Setup-tokens can expire; regenerate via `claude setup-token`
- Auth is per-agent; new agents do not inherit credentials
- The `extended-cache-ttl-2025-04-11` beta flag is included automatically

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } }
}
```

---

### OpenAI

**Auth Options:**

| Method | Setup Command | Best For |
|--------|---------------|----------|
| API Key | `openclaw onboard --openai-api-key "$OPENAI_API_KEY"` | Direct API billing |
| ChatGPT OAuth | `openclaw onboard --auth-choice openai-codex` | Subscription users |
| ChatGPT OAuth (alt) | `openclaw models auth login --provider openai-codex` | Subscription users |

**Quirks:**
- Codex cloud requires ChatGPT sign-in specifically (not API key)
- `openai` and `openai-codex` are separate providers with different auth flows
- OAuth uses PKCE flow with local callback on `http://127.0.0.1:1455/auth/callback`

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.1-codex" } } }
}
```

---

### OpenRouter

Unified API routing requests across multiple models through a single endpoint. OpenAI-compatible.

**Setup:** `openclaw onboard --auth-choice apiKey --token-provider openrouter`

**Model ID format uses triple-slash pattern:** `openrouter/<provider>/<model>`

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: { defaults: { model: { primary: "openrouter/anthropic/claude-sonnet-4-5" } } }
}
```

---

### Amazon Bedrock

**Credential priority (checked in order):**

1. `AWS_BEARER_TOKEN_BEDROCK` (bearer token)
2. `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` (explicit keys)
3. `AWS_PROFILE` (named profile)
4. Default AWS SDK credential chain (includes EC2 instance roles via IMDS)

**Required IAM permissions:**
- `bedrock:InvokeModel`
- `bedrock:InvokeModelWithResponseStream`
- `bedrock:ListFoundationModels` (discovery only)
- Or managed policy: `AmazonBedrockFullAccess`

**Model discovery:** Automatic, caches results for 1 hour (configurable via `refreshInterval`). Filters by streaming + text output support. Use `providerFilter` to filter by Bedrock provider (e.g., `anthropic`).

**Quirks:**
- Model access must be enabled in your AWS account/region
- Region defaults to `us-east-1` if not specified
- Credential detection does NOT check IMDS directly -- use `AWS_PROFILE=default` workaround on EC2
- Reasoning support varies by model
- Uses `auth: "aws-sdk"` (no `apiKey` field needed)
- Base URL: `https://bedrock-runtime.{region}.amazonaws.com`

```json5
{
  env: {
    AWS_ACCESS_KEY_ID: "AKIA...",
    AWS_SECRET_ACCESS_KEY: "...",
    AWS_REGION: "us-east-1"
  },
  agents: {
    defaults: {
      model: { primary: "amazon-bedrock/us.anthropic.claude-opus-4-6-v1:0" }
    }
  }
}
```

---

### Google Gemini

Multiple Google provider variants exist:

| Variant | Provider ID | Auth Method |
|---------|-------------|-------------|
| Gemini API | `google` | `GEMINI_API_KEY` |
| Vertex AI | `google-vertex` | Specific auth flow |
| Antigravity | `google-antigravity` | OAuth (Google account) |
| Gemini CLI | `google-gemini-cli` | CLI auth flow |

```json5
{
  env: { GEMINI_API_KEY: "..." },
  agents: { defaults: { model: { primary: "google/gemini-3-pro-preview" } } }
}
```

---

### Z.AI / GLM

Z.AI is the API platform for GLM models. GLM is a model family, not a company.

**Setup:** `openclaw onboard --auth-choice zai-api-key` or `openclaw onboard --zai-api-key "$ZAI_API_KEY"`

**Quirks:** GLM versions and availability can change; consult Z.AI documentation for current model roster.

```json5
{
  env: { ZAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "zai/glm-4.7" } } }
}
```

---

### Moonshot AI (Kimi)

**Two separate providers with non-interchangeable keys:**

| Provider | Provider ID | Env Variable | Setup Command |
|----------|-------------|--------------|---------------|
| Moonshot API | `moonshot` | `MOONSHOT_API_KEY` | `openclaw onboard --auth-choice moonshot-api-key` |
| Kimi Coding | `kimi-coding` | `KIMI_API_KEY` | `openclaw onboard --auth-choice kimi-code-api-key` |

**Moonshot models:**
- `kimi-k2.5` (default)
- `kimi-k2-0905-preview`
- `kimi-k2-turbo-preview`
- `kimi-k2-thinking` (reasoning enabled)
- `kimi-k2-thinking-turbo` (reasoning enabled)

**Kimi Coding models:** `k2p5`

**All models:** 256K context window, 8,192 max output, text-only input.

**Endpoint options:**
- International: `https://api.moonshot.ai/v1`
- China: `https://api.moonshot.cn/v1`

---

### MiniMax

**Auth Options:**

| Method | Setup |
|--------|-------|
| OAuth (recommended) | `openclaw plugins enable minimax-portal-auth` then `openclaw onboard --auth-choice minimax-portal` |
| API Key | Set `MINIMAX_API_KEY` env var |
| Local (LM Studio) | Manual JSON configuration |

**Models (case-sensitive IDs):**
- `minimax/MiniMax-M2.1` -- primary (200K context, 8,192 max output)
- `minimax/MiniMax-M2.1-lightning` -- faster variant, higher output costs

**Endpoint options:**
- Global: `api.minimax.io`
- China: `api.minimaxi.com`

**API modes:** `anthropic-messages` (preferred) and `openai-completions`

**Costs:** Input $0.015/1K, Output $0.060/1K, Cache read $0.002/1K, Cache write $0.010/1K

**Quirks:**
- Model IDs are case-sensitive -- `MiniMax-M2.1` not `minimax-m2.1`
- Lightning variant auto-routes during traffic; not directly selectable on Coding Plan
- Versions before 2026.1.12 may not auto-detect MiniMax provider
- Setup via `openclaw configure` > Model/auth > MiniMax M2.1

---

### Vercel AI Gateway

Unified API to access hundreds of models through a single endpoint. Anthropic Messages compatible.

**Setup:** `openclaw onboard --auth-choice ai-gateway-api-key`

**Model ID format:** `vercel-ai-gateway/<provider>/<model-name>`

**Daemon note:** When running as a background service (launchd/systemd), ensure the API key is exported to `~/.openclaw/.env` or shell environment.

```json5
{
  env: { AI_GATEWAY_API_KEY: "..." },
  agents: {
    defaults: {
      model: { primary: "vercel-ai-gateway/anthropic/claude-opus-4.6" }
    }
  }
}
```

---

### OpenCode Zen

Optional hosted model access path, currently in beta. Per-request billing.

**Setup:** `openclaw onboard --auth-choice opencode-zen`

**Env vars:** `OPENCODE_API_KEY` or `OPENCODE_ZEN_API_KEY`

```json5
{
  env: { OPENCODE_API_KEY: "..." },
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } }
}
```

---

### Qianfan (Baidu)

Baidu's MaaS platform with OpenAI-compatible unified API. Routes to many models behind a single endpoint.

**Setup:** `openclaw onboard --auth-choice qianfan-api-key`

**API key format:** `bce-v3/ALTAK-...` (obtained from Qianfan Console at `https://console.bce.baidu.com/qianfan/ais/console/apiKey`)

---

### Venice AI

Privacy-focused inference provider. Recommended for privacy-conscious deployments.

**Models:**
- `venice/llama-3.3-70b` (recommended default)
- `venice/claude-opus-45` (complex tasks)

---

## Local / Proxy Providers

For on-device inference or custom endpoints. Any OpenAI-compatible `/v1` endpoint works.

### Supported Local Runtimes

| Runtime | Default Endpoint | Notes |
|---------|------------------|-------|
| Ollama | `http://127.0.0.1:11434/v1` | Auto-detected by OpenClaw |
| LM Studio | `http://127.0.0.1:1234/v1` | Recommended local stack |
| vLLM | Custom | OpenAI-compatible |
| LiteLLM | Custom | OpenAI-compatible proxy |
| OAI-proxy | Custom | Any compatible gateway |

### Configuration Format

```json5
{
  models: {
    mode: "merge",       // "merge" keeps built-in providers alongside custom
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "minimax-m2.1-gs32",
            name: "MiniMax M2.1 GS32",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          }
        ]
      }
    }
  }
}
```

### Custom Proxy Example

```json5
{
  models: {
    mode: "merge",
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-responses",
        authHeader: true,
        headers: { "X-Proxy-Region": "us-west" },
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            api: "openai-responses",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 32000,
          }
        ]
      }
    }
  }
}
```

### Custom Provider Fields

| Field | Required | Description |
|-------|----------|-------------|
| `baseUrl` | Yes | HTTP(S) endpoint URL |
| `apiKey` | Yes | API key or local identifier |
| `api` | Yes | API type: `openai-responses`, `anthropic-messages`, `openai-completions`, `bedrock-converse-stream` |
| `authHeader` | No | Include auth header in requests |
| `headers` | No | Additional HTTP headers |
| `models` | Yes | Array of model definitions |
| `models[].id` | Yes | Model identifier |
| `models[].contextWindow` | No | Context window in tokens |
| `models[].maxTokens` | No | Max output tokens |
| `models[].reasoning` | No | Supports reasoning/thinking |
| `models[].input` | No | Input types (e.g., `["text"]`) |
| `models[].cost` | No | Cost per 1K tokens: `{ input, output, cacheRead, cacheWrite }` |

### Local Model Security Warnings

- Aggressively quantized or small checkpoints raise prompt-injection risk
- Local models bypass provider-side safety filters
- Always use the largest / full-size model variant you can run
- Mitigation: keep agents narrow, enable compaction

---

## Auth Profile Management

### CLI Commands

```bash
# Check status of all providers and auth
openclaw models status
openclaw models status --json       # Machine-readable output
openclaw models status --probe      # Live auth test (may consume tokens)

# List available models
openclaw models list

# Login to a provider
openclaw models auth login --provider <id>

# Setup-token flow (Anthropic)
openclaw models auth setup-token --provider anthropic

# Paste pre-generated token
openclaw models auth paste-token --provider anthropic

# Add auth profile
openclaw models auth add

# Set default model
openclaw models set <model-or-alias>

# List aliases and fallbacks
openclaw models aliases list
openclaw models fallbacks list
```

### Auth Profile Storage

| Location | Purpose |
|----------|---------|
| `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` | Primary (per-agent) |
| `~/.openclaw/agent/auth-profiles.json` | Legacy (shared) |
| `~/.openclaw/credentials/oauth.json` | OAuth import |

### Profile ID Formats

- API keys: `provider:default` (e.g., `anthropic:default`)
- OAuth accounts: `provider:<email>` (e.g., `google-antigravity:user@gmail.com`)

### Multi-Profile Configuration

```json5
{
  auth: {
    profiles: {
      "anthropic:me@example.com": { provider: "anthropic", mode: "oauth", email: "me@example.com" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:default": { provider: "openai", mode: "api_key" },
      "openai-codex:default": { provider: "openai-codex", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:me@example.com", "anthropic:work"],
      openai: ["openai:default"],
      "openai-codex": ["openai-codex:default"],
    },
  }
}
```

### OAuth with API Key Failover

```json5
{
  auth: {
    profiles: {
      "anthropic:subscription": { provider: "anthropic", mode: "oauth", email: "me@example.com" },
      "anthropic:api": { provider: "anthropic", mode: "api_key" },
    },
    order: {
      anthropic: ["anthropic:subscription", "anthropic:api"],
    },
  }
}
```

### Profile Selection Order

When multiple profiles exist for a provider, OpenClaw selects based on:

1. **Explicit configuration:** `auth.order[provider]` if set
2. **Configured profiles:** Listed in `auth.profiles`
3. **Stored profiles:** All entries in `auth-profiles.json`

**Round-robin default logic (without explicit ordering):**
- Primary sort: Profile type (OAuth before API keys)
- Secondary sort: "Oldest first" based on `usageStats.lastUsed`
- Disabled/cooldown profiles: Moved to end, ordered by soonest expiry

### Session Stickiness

OpenClaw pins the chosen auth profile per session to maintain provider cache efficiency. The pinned profile persists until:
- Session resets (`/new` or `/reset`)
- Compaction completes
- Profile enters cooldown/disabled state

**User overrides:** `/model Opus@anthropic:work` locks to that specific profile for the session and skips auto-rotation on failures.

**Auto-pinned profiles** are tried first but may rotate on rate limits/timeouts. If user-pinned profiles fail and fallbacks exist, OpenClaw switches models rather than profiles.

---

## Fallback Configuration

### Two-Stage Failover Process

OpenClaw implements failover in sequential stages:

1. **Auth Profile Rotation:** Cycles through available credentials within the same provider
2. **Model Fallback:** Advances to the next model in `agents.defaults.model.fallbacks`

### Configuration

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-5",
        fallbacks: ["anthropic/claude-opus-4-6", "openai/gpt-5.2"]
      }
    }
  }
}
```

### Anthropic + MiniMax Fallback Example

```json5
{
  auth: {
    profiles: {
      "anthropic:subscription": { provider: "anthropic", mode: "oauth", email: "user@example.com" },
      "anthropic:api": { provider: "anthropic", mode: "api_key" },
    },
    order: {
      anthropic: ["anthropic:subscription", "anthropic:api"],
    },
  },
  models: {
    providers: {
      minimax: {
        baseUrl: "https://api.minimax.io/anthropic",
        api: "anthropic-messages",
        apiKey: "${MINIMAX_API_KEY}",
      },
    },
  },
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.1"],
      }
    }
  }
}
```

### Model Aliases

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "anthropic/claude-sonnet-4-5": { alias: "sonnet" },
        "openai/gpt-5.2": { alias: "gpt" },
      }
    }
  }
}
```

### Fallback Triggers

Model fallback advances occur **only** for:
- Authentication failures
- Rate limits
- Timeouts (after exhausting profile rotation)

Other error types (e.g., invalid request, content filtering) do NOT trigger model fallback.

---

## Rate Limiting and Cooldown Behavior

### Cooldown Triggers

Cooldowns activate for:
- Authentication failures
- Rate-limiting errors
- Timeouts resembling rate limits
- Format/validation errors

### Exponential Backoff Schedule

| Attempt | Cooldown Duration |
|---------|-------------------|
| 1st | 1 minute |
| 2nd | 5 minutes |
| 3rd | 25 minutes |
| 4th+ | 1 hour (maximum) |

Stored in `usageStats`:
```json
{
  "cooldownUntil": 1736160600000,
  "errorCount": 2
}
```

### Billing Disables

Credit/billing failures trigger longer-term disables (not temporary cooldowns):

| Parameter | Value |
|-----------|-------|
| Initial backoff | 5 hours |
| Escalation | Doubles per subsequent failure |
| Maximum | 24 hours |
| Reset window | 24 hours without failures |

Stored as:
```json
{
  "disabledUntil": 1736178000000,
  "disabledReason": "billing"
}
```

### Cooldown Configuration Keys

| Setting | Description |
|---------|-------------|
| `auth.cooldowns.billingBackoffHours` | Provider-agnostic billing backoff |
| `auth.cooldowns.billingBackoffHoursByProvider` | Per-provider overrides |
| `auth.cooldowns.billingMaxHours` | Ceiling for billing disable duration |
| `auth.cooldowns.failureWindowHours` | Window for backoff counter reset |

---

## Hybrid Deployment Pattern

Use `models.mode: "merge"` to maintain hosted fallbacks alongside local primaries:

```json5
{
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [{
          id: "minimax-m2.1-gs32",
          contextWindow: 196608,
          maxTokens: 8192,
          cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }
        }]
      }
    }
  },
  agents: {
    defaults: {
      model: {
        primary: "lmstudio/minimax-m2.1-gs32",
        fallbacks: ["anthropic/claude-opus-4-6"]
      }
    }
  }
}
```

---

## Quick Reference: Onboarding Commands

| Provider | Command |
|----------|---------|
| Anthropic (API key) | `openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"` |
| Anthropic (setup-token) | `openclaw models auth setup-token --provider anthropic` |
| OpenAI (API key) | `openclaw onboard --openai-api-key "$OPENAI_API_KEY"` |
| OpenAI Codex (OAuth) | `openclaw onboard --auth-choice openai-codex` |
| OpenRouter | `openclaw onboard --auth-choice apiKey --token-provider openrouter` |
| Amazon Bedrock | Configure `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` + `AWS_REGION` env vars |
| Google Gemini | Set `GEMINI_API_KEY` env var |
| Z.AI / GLM | `openclaw onboard --auth-choice zai-api-key` |
| Moonshot | `openclaw onboard --auth-choice moonshot-api-key` |
| Kimi Coding | `openclaw onboard --auth-choice kimi-code-api-key` |
| MiniMax (OAuth) | `openclaw plugins enable minimax-portal-auth` then `openclaw onboard --auth-choice minimax-portal` |
| MiniMax (API key) | Set `MINIMAX_API_KEY` env var |
| Vercel AI Gateway | `openclaw onboard --auth-choice ai-gateway-api-key` |
| OpenCode Zen | `openclaw onboard --auth-choice opencode-zen` |
| Qianfan | `openclaw onboard --auth-choice qianfan-api-key` |
| Venice AI | Set up via `openclaw onboard` |
| Non-interactive (any) | `openclaw onboard --non-interactive --mode local --auth-choice <choice>` |

---

## Troubleshooting

```bash
# Full status check
openclaw models status

# JSON output for scripting
openclaw models status --json

# Live probe (may consume tokens)
openclaw models status --probe

# Check gateway health
openclaw doctor

# View cooldown/unavailable profiles
# Check auth.unusableProfiles in status output

# Scan for available models
openclaw models scan
```

### Common Issues

| Problem | Solution |
|---------|----------|
| "Unknown model" error | Upgrade to latest version, run `openclaw configure`, or manually add `models.providers` block |
| Setup-token expired | Regenerate via `claude setup-token` and re-paste |
| Per-agent auth missing | Run onboarding per agent or copy `auth-profiles.json` between agent dirs |
| EC2 IMDS credentials not detected | Set `AWS_PROFILE=default` as workaround |
| MiniMax not auto-detected | Requires version 2026.1.12+; run `openclaw configure` |
| Daemon missing API key | Export to `~/.openclaw/.env` or shell environment for launchd/systemd |

---

## Continuous Improvement

**Always verify with current docs:** Check the bundled docs at the openclaw installation path, or fetch from https://docs.openclaw.ai/ to check for updates.

When using OpenClaw and discovering undocumented features, corrections, or better practices:
1. Update this file at `~/.claude/skills/openclaw-guide/references/model-providers.md`
2. Add new sections for newly discovered features
3. Correct any outdated or inaccurate information
4. Add practical examples from real usage

---
*Source: https://docs.openclaw.ai/ -- Last updated 2026-02-08*
*Skill: ~/.claude/skills/openclaw-guide*
