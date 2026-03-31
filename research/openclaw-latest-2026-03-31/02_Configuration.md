# 02 — OpenClaw Configuration

> **Source notes:** Based on workspace research notes dated 2026-03-30.
> Live docs: [docs.openclaw.ai](https://docs.openclaw.ai) [unverified — not fetched this session]

<!-- IMAGE: openclaw.json file structure diagram showing key sections -->

---

## Config File Location

```
~/.openclaw/openclaw.json
```

This is the primary configuration file. Set strict permissions immediately after setup:

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
```

---

## openclaw.json — Full Structure

```json5
{
  // Environment variables (API keys go here, not hardcoded below)
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    FIRECRAWL_API_KEY: "fc-..."
  },

  // Gateway settings
  gateway: {
    port: 18789,
    bind: "loopback"      // SECURITY: bind to 127.0.0.1 only
  },

  // Agent model routing
  agents: {
    defaults: {
      model: {
        primary: "openrouter/anthropic/claude-sonnet-4-6"
      }
    }
  },

  // Full model routing config
  models: {
    primary:   "openrouter/anthropic/claude-sonnet-4-6",
    subagent:  "openrouter/deepseek/deepseek-v3.2",
    heartbeat: "openrouter/google/gemini-2.5-pro"
  },

  // Provider config
  providers: {
    openrouter: {
      apiKey: "${OPENROUTER_API_KEY}"    // reference env var
    }
  },

  // Channel config
  channels: {
    telegram: {
      enabled: true,
      botToken: "${TELEGRAM_BOT_TOKEN}",
      dmPolicy: "allowlist",
      allowFrom: ["YOUR_NUMERIC_TELEGRAM_ID"]
    }
  }
}
```

> Note: OpenClaw uses JSON5 format (comments allowed, trailing commas ok). Do not use standard JSON parsers to edit it.

---

## OpenRouter Integration

### Why OpenRouter?

OpenRouter provides a single API key that routes to all major LLMs — Anthropic, OpenAI, Google, DeepSeek, and 200+ others. This is the recommended provider for OpenClaw.

- One key for all models
- Native OpenClaw integration
- Cost tracking per model
- Automatic fallback routing [unverified]

### Setup

1. Get your key at [openrouter.ai](https://openrouter.ai)
2. Run the auth command:

```bash
openclaw onboard --auth-choice apiKey --token-provider openrouter --token "sk-or-YOUR_KEY"
```

Or add directly to `openclaw.json`:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-..."
  },
  providers: {
    openrouter: {
      apiKey: "${OPENROUTER_API_KEY}"
    }
  }
}
```

### Model Routing Strategy

| Role | Model String | Cost/1M tokens | When to Use |
|------|-------------|----------------|-------------|
| Orchestrator / synthesis | `openrouter/anthropic/claude-sonnet-4-6` | $3 in / $15 out | Final reports, complex reasoning |
| Web research + search | `openrouter/openai/gpt-4o` | $2.50 in / $10 out | When built-in web search needed |
| Mid-tier analysis | `openrouter/google/gemini-2.5-pro` | $1.25 in / $5 out | Analysis tasks, structured extraction |
| Bulk crawl / summarize | `openrouter/deepseek/deepseek-v3.2` | $0.28 in / $0.42 out | 90% of tasks — cheapest workhorse |

**Key cost principle:** Route 90% of tasks (scraping, summarizing, bulk data) to DeepSeek. Escalate to Claude Sonnet only for final synthesis.

---

## Workspace Files

The onboarding wizard creates `~/.openclaw/workspace/` with these files. Customize all of them for your use case.

### SOUL.md

Injected as system prompt on every session. Defines the agent's persistent identity, tone, and operational style.

```markdown
# Agent Identity

You are a financial research agent running on a personal Mac Mini M4.
Your job is to gather data, analyze market conditions, and produce
structured investment research reports.

You are:
- Precise and systematic
- Source everything with citations
- Explicit about uncertainty (mark estimates clearly)
- Focused on actionable insights, not speculation

You do not speculate without supporting data.
```

### USER.md

Your personal context — the agent reads this to understand who it's working for.

```markdown
# User Profile

- Timezone: [YOUR TIMEZONE]
- Primary goal: Financial research and strategy automation
- Preferred report format: Structured markdown with source citations
- Delivery channel: Telegram
- Research focus: [YOUR MARKETS/SECTORS]
```

### AGENTS.md

Workflow definitions and operational rules. Define your research pipeline stages, sub-agent roles, and handoff protocols here. See `03_Multi_Agent_Setup.md` for full AGENTS.md examples.

### HEARTBEAT.md

Scheduled autonomous task configuration. Example for financial research:

```markdown
# Scheduled Tasks

## Daily — 7:00 AM
- Pull overnight market news from 3 sources via Firecrawl
- Summarize key developments affecting watchlist
- Deliver morning briefing to Telegram

## Weekly — Sunday 8:00 AM
- Broader sector analysis across watchlist
- Compile weekly strategy summary
- Deliver to Telegram

## On-demand triggers
- "research [ticker]" → full research pipeline
- "brief me" → quick morning summary
```

> Best practice: Version-control `~/.openclaw/workspace/` in a private Git repo. This makes your entire agent configuration reproducible.

---

## Daemon Management on macOS

OpenClaw installs as a LaunchAgent (not LaunchDaemon) — it runs under your user account at login.

### Plist Location

```
~/Library/LaunchAgents/com.openclaw.gateway.plist
```

### Standard Commands

```bash
openclaw gateway status     # is it running?
openclaw gateway start      # start daemon
openclaw gateway stop       # stop daemon
openclaw gateway install    # reinstall LaunchAgent plist if missing
```

### Known KeepAlive Bug on macOS

`openclaw gateway restart` has a known bug on macOS caused by a launchd KeepAlive race condition. When called, it leaves the service in a broken/zombie state.

**Workaround — always use stop + start:**

```bash
# CORRECT: stop then start separately
openclaw gateway stop
openclaw gateway start

# WRONG: do not use restart
# openclaw gateway restart   ← broken on macOS
```

### Fix the KeepAlive Behavior

To prevent launchd from auto-respawning after a manual stop, edit the plist:

```bash
nano ~/Library/LaunchAgents/com.openclaw.gateway.plist
```

Find:
```xml
<key>KeepAlive</key>
<true/>
```

Replace with:
```xml
<key>KeepAlive</key>
<dict>
  <key>SuccessfulExit</key>
  <false/>
</dict>
```

Then reload the plist:
```bash
launchctl unload ~/Library/LaunchAgents/com.openclaw.gateway.plist
launchctl load ~/Library/LaunchAgents/com.openclaw.gateway.plist
```

### Manual launchctl Commands (Fallback)

```bash
# Check if openclaw is registered with launchd
launchctl list | grep openclaw

# Reload if missing
launchctl load ~/Library/LaunchAgents/com.openclaw.gateway.plist

# Stop/start via launchctl directly
launchctl stop ~/Library/LaunchAgents/com.openclaw.gateway.plist
launchctl start ~/Library/LaunchAgents/com.openclaw.gateway.plist
```

---

## Dashboard

OpenClaw ships a browser-based dashboard for monitoring and config:

```bash
openclaw dashboard
# Opens: http://localhost:18789
```

---

## Sources

- Workspace notes: `research/setup-guide-2026-03-30.md`
- Workspace notes: `research/getting-started-openclaw-2026-03-30.md`
- [docs.openclaw.ai/providers/openrouter](https://docs.openclaw.ai/providers/openrouter) [link unverified this session]
- [OpenRouter + OpenClaw Integration](https://openrouter.ai/docs/guides/guides/openclaw-integration) [link unverified this session]
- [github.com/TechNickAI/openclaw-config](https://github.com/TechNickAI/openclaw-config) [link unverified this session]
