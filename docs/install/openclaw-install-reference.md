# OpenClaw Install Reference

Hardware decisions, installation steps, and configuration reference for the Mac Mini M4 setup.

---

## What is OpenClaw?

OpenClaw is a self-hosted personal AI agent gateway ([github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)) that runs as a persistent daemon connecting LLMs to your messaging apps and local system. Key capabilities:

- Responds via WhatsApp, Telegram, iMessage, Discord, Slack, and 20+ other platforms
- Executes shell commands, manages files, automates browser tasks
- Runs scheduled tasks (cron), handles webhooks, controls a Chrome/Chromium instance
- Extends via "skills" from ClawHub (public skill registry)

**Software requirements:** Node.js 22+, macOS/Linux/WSL2.

---

## Hardware Decision: Mac Mini M4, 16GB RAM, 512GB SSD

**Status: PURCHASED**

### Why M4 over M1 used

- M1 used 16GB costs ~$500–580 — not much cheaper
- M4 is 1.8x faster CPU, lower power draw, longer software support
- At similar price points, M4 new wins

### Why 16GB is enough (no local LLMs)

- OpenClaw daemon: ~300MB
- macOS idle: ~2–3GB
- 4–5 concurrent agents (with browser): ~6–10GB
- 16GB covers this comfortably

### Why 512GB internal (not 256GB + external)

- Avoids external drive complexity
- Internal NVMe is faster for research outputs, crawled data, and logs
- Clean single-machine setup
- Can add external 1TB SSD later if storage fills up

### What was ruled out

| Option | Reason |
|--------|--------|
| Local LLMs | Competes with agent RAM, cloud APIs cheap enough |
| M1 Mac Mini | Similar used price to M4 new, slower, less future-proof |
| M4 Pro 48GB | Only needed for 32B+ local models — overkill |
| M4 24GB | $400 more, only useful for local models |
| Gemini 3.1 Pro | Preview only, unstable, more expensive than 2.5 Pro |
| External SSD (now) | 512GB internal is sufficient to start |

---

## Use Case

Financial research and strategy automation using **multiple concurrent agents** doing web crawling, data gathering, and analysis — all routed to cloud LLMs. No local models. Results delivered via Telegram.

---

## LLM Stack (Cloud APIs Only)

Accessed via **direct API keys** (no OpenRouter middleman — avoids ~8% markup):

| Priority | Model | Role | Cost/1M tokens |
|----------|-------|------|----------------|
| 1st | DeepSeek V3.2 | Bulk crawling & summarizing | $0.28 in / $0.42 out |
| 2nd | Gemini 2.5 Pro | Mid-tier analysis | $1.25 in / $5 out |
| 3rd | GPT-4o | Web research (built-in search) | $2.50 in / $10 out |
| 4th | Claude Sonnet 4.6 | Final synthesis & strategy | $3 in / $15 out |

**Key rule:** Let DeepSeek handle 90% of tasks. Use Claude only for final reports.

### Monthly Cost Estimate

| Usage | Cost |
|-------|------|
| Light (5–10 tasks/day) | ~$55/mo |
| Medium (20–30 tasks/day) | ~$161/mo |
| Heavy (50+ tasks/day) | ~$320/mo |

---

## Tools Chosen

| Tool | Purpose |
|------|---------|
| OpenClaw | Self-hosted AI agent gateway |
| Direct API keys | Unified access to all 4 LLMs (no middleman) |
| Firecrawl | Remote web scraping with browser support |
| Telegram | First messaging channel (easiest to set up) |

---

## Prerequisites

### 1. Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Verify:
```bash
brew --version
```

### 2. Node.js 24 (via Homebrew)

```bash
brew install node@24
```

**Requirement:** Node.js 22.14+ minimum (ideally 24.x)

Verify:
```bash
node --version
```

### 3. pnpm (optional but recommended)

```bash
brew install pnpm
```

Or use npm if you prefer:
```bash
npm install -g pnpm
```

---

## Installation

### Option A: One-Line Installer (Recommended)

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

This automatically:
- Installs Homebrew if not present
- Installs Node.js 24 via Homebrew
- Installs the OpenClaw CLI globally
- Runs `openclaw doctor` as a post-install check

Verify:
```bash
openclaw --version
```

### Option B: Manual Install via Package Manager

```bash
pnpm add -g openclaw@latest
```

Or with npm:
```bash
npm install -g openclaw@latest
```

Verify:
```bash
openclaw --version
```

### Which method to choose?

| Method | Best For | Complexity |
|--------|----------|-----------|
| **One-line installer** | Fresh Mac, fastest setup | Easy |
| **pnpm/npm** | Existing Node setup, package manager preference | Easy |
| Docker | Isolated/sandboxed environment | Medium |
| Build from source | Developers, latest features | Hard |

**For this setup: the one-line installer is the right choice.** Docker adds overhead through a Linux VM layer on macOS — not worth it for a personal machine. The installer gives the best performance and simplest setup.

---

## Onboarding Wizard

This is the key step — the wizard sets up everything:

```bash
openclaw onboard --install-daemon
```

The `--install-daemon` flag is important — it installs OpenClaw as a **macOS LaunchAgent** so it:
- Starts automatically on every boot
- Keeps running after you close Terminal
- Runs silently in the background 24/7

### What the wizard asks you

1. **LLM provider** — Choose "Direct API keys" and paste each provider's key
2. **Auth token** — generates a secure token for your gateway
3. **Channel** — which messaging app to connect (choose Telegram first)
4. **Skills** — optional tools to install (can skip and add later)
5. **Workspace location** — Accept the default (`~/.openclaw/workspace`)

---

## Configuration

### Direct API Keys Setup

Edit the config file at `~/.openclaw/openclaw.json`:

```json
{
  "env": {
    "ANTHROPIC_API_KEY": "sk-ant-...",
    "OPENAI_API_KEY": "sk-...",
    "GOOGLE_API_KEY": "AIza...",
    "DEEPSEEK_API_KEY": "sk-..."
  },
  "models": {
    "primary": "anthropic/claude-sonnet-4-6",
    "subagent": "deepseek/deepseek-v3.2",
    "heartbeat": "google/gemini-2.5-pro"
  }
}
```

Get API keys directly from each provider — no OpenRouter middleman.

### LLM Role Assignments

Each sub-agent in a research pipeline gets its own model based on task type:

| Role | Model | Why |
|------|-------|-----|
| Orchestrator / synthesis | `anthropic/claude-sonnet-4-6` | Best reasoning for strategy and final reports |
| Web research + search | `openai/gpt-4o` | Strong tool use and search |
| Mid-tier analysis | `google/gemini-2.5-pro` | Good cost/capability for analysis tasks |
| Bulk crawl / summarize | `deepseek/deepseek-v3.2` | Cheapest — use for 90% of data gathering |

**Cost tip:** Route the majority of tasks to DeepSeek. Only escalate to Claude/GPT-4o for final synthesis.

---

## Telegram Channel Setup

Telegram is the easiest channel to connect — takes ~5 minutes.

### 1. Create your bot via BotFather

1. Open Telegram, search for **@BotFather** (verify blue checkmark)
2. Send `/newbot` — follow prompts, get a **bot token**
3. Copy your bot token (format: `123456789:ABCdef...`) — treat like a password

### 2. Configure in openclaw.json

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_BOT_TOKEN",
      "dmPolicy": "allowlist",
      "allowFrom": ["YOUR_NUMERIC_TELEGRAM_ID"]
    }
  }
}
```

Use `allowlist` policy (not `pairing`) — it's more durable across gateway restarts.

### 3. Find your numeric Telegram ID

Message your bot, then run:
```bash
openclaw logs --follow
```

Look for `from.id` in the output — that's your numeric ID.

### 4. Start chatting

Start a chat with your new bot in Telegram and send a message — OpenClaw responds.

---

## Skills & Firecrawl

### Install Firecrawl Skill (Web Research)

Firecrawl provides remote web scraping with a real browser, avoiding 403 errors and empty HTML responses.

```bash
clawhub install firecrawl
```

Firecrawl advantages for research pipelines:
- Routes through a real remote browser — avoids 403 errors and empty HTML responses
- No local Chromium RAM overhead
- Supports parallel sessions

### Configure Firecrawl API Key

Add your Firecrawl API key to `~/.openclaw/openclaw.json`:

```json
{
  "env": {
    "FIRECRAWL_API_KEY": "fc-..."
  }
}
```

Get a Firecrawl API key at [firecrawl.dev](https://firecrawl.dev).

### Security Warning

**Only install ClawHub skills you have personally reviewed.** Malicious skills with credential-harvesting payloads have been found in the registry.

---

## Workspace Files

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

## Daemon Control and Management

### Basic Commands

```bash
# Check status
openclaw gateway status

# Start
openclaw gateway start

# Stop
openclaw gateway stop

# View logs
openclaw logs --follow
```

### Important: The restart bug on macOS

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

### If the daemon mysteriously disappears

Check:
```bash
launchctl list | grep openclaw
```

If missing, reload manually:
```bash
launchctl load ~/Library/LaunchAgents/com.openclaw.gateway.plist
```

Or reinstall the LaunchAgent:
```bash
openclaw gateway install
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

## Security Hardening

`~/.openclaw/` is a high-privilege directory — treat it like a password vault.

### Immediate hardening steps

```bash
chmod 700 ~/.openclaw
chmod 600 ~/.openclaw/openclaw.json
openclaw security audit --fix
```

### Security Checklist

- [ ] `chmod 700 ~/.openclaw`
- [ ] `chmod 600 ~/.openclaw/openclaw.json`
- [ ] `openclaw security audit --fix`
- [ ] API keys stored via env vars, not hardcoded in config
- [ ] `openclaw.json` and `auth-profiles.json` excluded from any Git repos
- [ ] Telegram set to `dmPolicy: "allowlist"` with your numeric ID
- [ ] Gateway bound to loopback only (`gateway.bind: "loopback"`)
- [ ] Only install ClawHub skills you've personally reviewed
- [ ] Applied conditional KeepAlive fix to LaunchAgent plist
- [ ] Never share your auth token
- [ ] Keep `~/.openclaw/openclaw.json` permissions restricted

### Important Notes

- Never share your auth token
- Disable DMs from unknown users in channel settings
- **Do NOT use a Claude Pro subscription** to power OpenClaw — Anthropic's ToS prohibit it. You need a paid API key via direct provider access.

---

## Quick Reference

| Command | What it does |
|---------|-------------|
| `openclaw gateway status` | Check if daemon is running |
| `openclaw gateway start` | Start the daemon |
| `openclaw gateway stop` | Stop the daemon |
| `openclaw gateway stop && openclaw gateway start` | Restart (use this, NOT `restart`) |
| `openclaw gateway install` | Reinstall LaunchAgent plist if needed |
| `openclaw logs --follow` | Live log stream |
| `openclaw security audit` | Check for security issues |
| `openclaw doctor --fix` | Diagnose and fix common problems |
| `clawhub install <skill>` | Install a skill from the registry |
| `openclaw dashboard` | Open browser dashboard at localhost:18789 |

---

## Setup Order (Step by Step)

- [ ] 1. Install Homebrew
- [ ] 2. Install Node.js 24 via Homebrew
- [ ] 3. Install pnpm (optional)
- [ ] 4. Install OpenClaw (one-liner or pnpm)
- [ ] 5. Run `openclaw onboard --install-daemon`
- [ ] 6. Harden security: `chmod 700 ~/.openclaw && chmod 600 ~/.openclaw/openclaw.json`
- [ ] 7. Create Telegram bot via BotFather
- [ ] 8. Configure Telegram in `~/.openclaw/openclaw.json`
- [ ] 9. Get API keys from Anthropic, OpenAI, Google AI, DeepSeek
- [ ] 10. Configure API keys in `~/.openclaw/openclaw.json`
- [ ] 11. Get Firecrawl API key and configure it
- [ ] 12. Install Firecrawl skill: `clawhub install firecrawl`
- [ ] 13. Customize `~/.openclaw/workspace/SOUL.md`, `USER.md`, `AGENTS.md`
- [ ] 14. Apply KeepAlive fix to LaunchAgent plist
- [ ] 15. Run final security audit: `openclaw security audit`
- [ ] 16. Test — send first message via Telegram

---

## Community & Help

| Resource | Best for |
|----------|---------|
| [Official Docs](https://docs.openclaw.ai) | Reference, config options |
| [OpenClaw Discord](https://docs.openclaw.ai/channels/discord) | Technical questions, bugs |
| [r/openclaw](https://reddit.com/r/openclaw) | Use cases, tips, discussion |
| [GitHub Issues](https://github.com/openclaw/openclaw) | Bug reports, feature requests |
| [GitHub Discussions](https://github.com/openclaw/openclaw/discussions) | Setup help |

---

## Sources

- [OpenClaw Official Docs — Install](https://docs.openclaw.ai/install)
- [OpenClaw macOS Install Guide — AlexHost](https://alexhost.com/faq/how-to-install-openclaw-on-macos/)
- [How to Install OpenClaw on Mac — Medium/Zilliz](https://medium.com/@zilliz_learn/how-to-install-and-run-openclaw-previously-clawdbot-moltbot-on-mac-9cb6adb64eef)
- [OpenClaw Install Guide 2026 — Boilerplate Hub](https://boilerplatehub.com/blog/openclaw-install)
- [How to Run OpenClaw: Daemon & TUI — Dextra Labs](https://dextralabs.com/blog/how-to-run-openclaw/)
- [docs.openclaw.ai/providers/openrouter](https://docs.openclaw.ai/providers/openrouter) — OpenRouter integration
- [docs.openclaw.ai/channels/telegram](https://docs.openclaw.ai/channels/telegram) — Telegram setup
- [docs.openclaw.ai/gateway/security](https://docs.openclaw.ai/gateway/security) — Security hardening
- [docs.firecrawl.dev/developer-guides/openclaw](https://docs.firecrawl.dev/developer-guides/openclaw) — Firecrawl integration
- [github.com/TechNickAI/openclaw-config](https://github.com/TechNickAI/openclaw-config) — Community config example
- [github.com/digitalknk/openclaw-runbook](https://github.com/digitalknk/openclaw-runbook) — Operational runbook
