# OpenClaw Setup Guide — Mac Mini M4 (Financial Research)

## Overview

This guide covers a complete from-scratch setup of OpenClaw on a fresh Mac Mini M4 (16GB RAM, 512GB SSD) for financial research and multi-agent workflows. All inference is via cloud APIs through OpenRouter — no local models.

---

## Before You Start

**What you'll need:**
- Mac Mini M4 connected, logged in, with internet access
- API keys for each provider: Anthropic, OpenAI, Google AI, DeepSeek (direct — no OpenRouter)
- Telegram account (for the messaging channel)
- Firecrawl API key ([firecrawl.dev](https://firecrawl.dev)) for web scraping

**Do NOT use a Claude Pro subscription** to power OpenClaw — Anthropic's ToS prohibit it. You need a paid API key via OpenRouter or directly.

---

## Phase 1 — Base Install

### 1. Run the one-line installer

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

This automatically:
- Installs Homebrew if not present
- Installs Node.js 24 via Homebrew (required: 22.14+ minimum)
- Installs the OpenClaw CLI globally
- Runs `openclaw doctor` as a post-install check

Verify it worked:
```bash
node --version    # should be 22.14+ (ideally 24.x)
openclaw --version
```

### 2. Run the onboarding wizard

```bash
openclaw onboard --install-daemon
```

During the wizard:
- **LLM provider:** Choose OpenRouter, paste your API key
- **Channel:** Choose Telegram
- **Workspace location:** Accept the default (`~/.openclaw/workspace`)

The `--install-daemon` flag installs OpenClaw as a LaunchAgent so it starts automatically on login.

### 3. Harden security immediately

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
openclaw security audit --fix
```

---

## Phase 2 — Configure OpenRouter + LLM Stack

Configure direct API keys in `~/.openclaw/openclaw.json`:

```json5
{
  env: {
    ANTHROPIC_API_KEY: "sk-ant-...",
    OPENAI_API_KEY: "sk-...",
    GOOGLE_API_KEY: "AIza...",
    DEEPSEEK_API_KEY: "sk-..."
  },
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6"
      }
    }
  }
}
```

### LLM role assignments

Each sub-agent in a research pipeline gets its own model based on task type:

| Role | Model | Why |
|------|-------|-----|
| Orchestrator / synthesis | `anthropic/claude-sonnet-4-6` | Best reasoning for strategy and final reports |
| Web research + search | `openai/gpt-4o` | Strong tool use and search |
| Mid-tier analysis | `google/gemini-2.5-pro` | Good cost/capability for analysis tasks |
| Bulk crawl / summarize | `deepseek/deepseek-v3.2` | Cheapest — use for 90% of data gathering |

Cost tip: Route the majority of tasks to DeepSeek. Only escalate to Claude/GPT-4o for final synthesis.

---

## Phase 3 — Telegram Setup

### 1. Create your bot via BotFather

1. Open Telegram, search for `@BotFather` (verify blue checkmark)
2. Send `/newbot` and follow prompts
3. Copy your bot token (format: `123456789:ABCdef...`) — treat like a password

### 2. Configure in openclaw.json

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "YOUR_BOT_TOKEN",
      dmPolicy: "allowlist",
      allowFrom: ["YOUR_NUMERIC_TELEGRAM_ID"]
    }
  }
}
```

Use `allowlist` policy (not `pairing`) — it's more durable across gateway restarts.

**To find your numeric Telegram ID:** Message your bot, then run `openclaw logs --follow` and look for `from.id` in the output.

---

## Phase 4 — Install Firecrawl Skill

> **Note:** Crawl4AI has no official OpenClaw skill. Firecrawl is the documented path for web scraping within OpenClaw.

```bash
clawhub install firecrawl
```

Firecrawl advantages for research pipelines:
- Routes through a real remote browser — avoids 403 errors and empty HTML responses
- No local Chromium RAM overhead
- Supports parallel sessions

Add your Firecrawl API key to env in `openclaw.json`:

```json5
{
  env: {
    FIRECRAWL_API_KEY: "fc-..."
  }
}
```

**Security warning:** Only install ClawHub skills you have personally reviewed. Malicious skills with credential-harvesting payloads have been found in the registry.

---

## Phase 5 — Workspace Files

The wizard creates `~/.openclaw/workspace/` with these key files. Customize them for your use case.

### SOUL.md
Injected on every session — defines the agent's persistent identity and behavior. Set the tone and operational style here.

```markdown
You are a financial research agent. Your job is to gather data, analyze market conditions,
and produce structured investment research reports. You are precise, source everything,
and flag uncertainty explicitly. You do not speculate without data.
```

### USER.md
Your context — the agent reads this to understand who it's working for.

```markdown
- Timezone: [YOUR TIMEZONE]
- Primary goal: Financial research and strategy automation
- Preferred report format: Structured markdown with sources cited
- Delivery: Telegram
```

### AGENTS.md
Your workflow definitions and operational rules. Define your research pipeline stages here.

### HEARTBEAT.md
Your scheduled/autonomous task schedule. Example for financial research:

```markdown
## Daily (7:00 AM)
- Pull overnight market news from 3 sources via Firecrawl
- Summarize key developments affecting watchlist
- Deliver briefing to Telegram

## Weekly (Sunday 8:00 AM)
- Broader sector analysis across watchlist
- Compile weekly strategy summary
- Deliver to Telegram
```

**Best practice:** Version-control `~/.openclaw/workspace/` in a private Git repo. This makes your agent config reproducible if you ever need to reinstall.

---

## Phase 6 — Daemon Hardening

### Fix the broken restart behavior on macOS

`openclaw gateway restart` has a known bug on macOS (launchd KeepAlive race condition). Until it's patched, always use:

```bash
openclaw gateway stop
openclaw gateway start
```

Never use `openclaw gateway restart` — it leaves the service in a broken state.

### Apply the KeepAlive fix

Edit the LaunchAgent plist:

```bash
nano ~/Library/LaunchAgents/com.openclaw.gateway.plist
```

Find the `<key>KeepAlive</key><true/>` block and replace with:

```xml
<key>KeepAlive</key>
<dict>
  <key>SuccessfulExit</key>
  <false/>
</dict>
```

This prevents launchd from auto-respawning after a manual stop.

### Useful daemon commands

```bash
openclaw gateway status    # check if running
openclaw gateway start
openclaw gateway stop
openclaw gateway install   # reinstall LaunchAgent plist if needed
```

If the daemon mysteriously disappears, check:
```bash
launchctl list | grep openclaw
# If missing, reload manually:
launchctl load ~/Library/LaunchAgents/com.openclaw.gateway.plist
```

---

## Financial Research Pipeline Architecture

Each pipeline stage runs as a discrete sub-agent with its own workspace and model assignment:

```
User sends research request via Telegram
  │
  ▼
Claude (Orchestrator)
  Scope definition + subtopic mapping
  │
  ├─► DeepSeek (Data Fetcher) ──── Firecrawl scrape → markdown files
  ├─► Gemini (Metrics Analyst) ─── analyze data → markdown files
  ├─► Gemini (Risk Assessor) ───── risk assessment → markdown files
  └─► Claude (Thesis Writer) ───── synthesis → draft report
  │
  ▼
Claude (Orchestrator)
  Compiles master report
  │
  ▼
Delivered via Telegram
```

Route 90% of tasks (scraping, summarizing, bulk analysis) to DeepSeek. Escalate to Claude only for final synthesis.

---

## Security Checklist

- [ ] `chmod 700 ~/.openclaw`
- [ ] `chmod 600 ~/.openclaw/openclaw.json`
- [ ] `openclaw security audit --fix`
- [ ] API keys stored via env vars, not hardcoded in config
- [ ] `openclaw.json` and `auth-profiles.json` excluded from any Git repos
- [ ] Telegram set to `dmPolicy: "allowlist"` with your numeric ID
- [ ] Gateway bound to loopback only (`gateway.bind: "loopback"`)
- [ ] Only install ClawHub skills you've personally reviewed
- [ ] Applied conditional KeepAlive fix to LaunchAgent plist

---

## Quick Reference

| Command | What it does |
|---------|-------------|
| `openclaw gateway status` | Check if daemon is running |
| `openclaw gateway stop && openclaw gateway start` | Restart (use this, NOT `restart`) |
| `openclaw logs --follow` | Live log stream |
| `openclaw security audit` | Check for security issues |
| `openclaw doctor --fix` | Diagnose and fix common problems |
| `clawhub install <skill>` | Install a skill from the registry |
| `openclaw dashboard` | Open browser dashboard at localhost:18789 |

---

## Key Sources

- [docs.openclaw.ai](https://docs.openclaw.ai) — official docs
- [docs.openclaw.ai/providers/openrouter](https://docs.openclaw.ai/providers/openrouter) — OpenRouter integration
- [docs.openclaw.ai/channels/telegram](https://docs.openclaw.ai/channels/telegram) — Telegram setup
- [docs.openclaw.ai/gateway/security](https://docs.openclaw.ai/gateway/security) — security hardening
- [docs.firecrawl.dev/developer-guides/openclaw](https://docs.firecrawl.dev/developer-guides/openclaw) — Firecrawl integration
- [github.com/TechNickAI/openclaw-config](https://github.com/TechNickAI/openclaw-config) — community config example
- [github.com/digitalknk/openclaw-runbook](https://github.com/digitalknk/openclaw-runbook) — operational runbook
