# Getting Started with OpenClaw on Mac Mini M4

## Installation Options

There are 3 ways to install OpenClaw. Choose one:

| Method | Best For | Complexity |
|--------|----------|-----------|
| **npm (recommended)** | Personal Mac, fastest setup | Easy |
| Docker | Isolated/sandboxed environment | Medium |
| Build from source | Developers, latest features | Hard |

**For our setup: npm is the right choice.** Docker adds overhead through a Linux VM layer on macOS — not worth it for a personal machine. npm gives the best performance and simplest setup.

---

## Prerequisites

### 1. Homebrew
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
Verify: `brew --version`

### 2. Node.js 24 (via Homebrew)
```bash
brew install node@24
```
Verify: `node --version` (should show v24.x.x)

### 3. pnpm (faster than npm, recommended)
```bash
brew install pnpm
```
Or use npm if you prefer:
```bash
npm install -g pnpm
```

---

## Install OpenClaw

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

---

## Run Onboarding

This is the key step — the wizard sets up everything:

```bash
openclaw onboard --install-daemon
```

The `--install-daemon` flag is important — it installs OpenClaw as a **macOS launchd service** so it:
- Starts automatically on every boot
- Keeps running after you close Terminal
- Runs silently in the background 24/7

### What the wizard asks you:
1. **Auth token** — generates a secure token for your gateway
2. **Gateway settings** — port and bind address (defaults are fine)
3. **Channel** — which messaging app to connect (choose Telegram first)
4. **Skills** — optional tools to install (can skip and add later)

---

## Daemon Control (macOS)

After setup, control the daemon with:

```bash
# Check status
openclaw status

# Stop
openclaw stop

# Start
openclaw start

# Restart
openclaw restart
```

Or via launchctl directly:
```bash
launchctl stop ~/Library/LaunchAgents/com.openclaw.gateway.plist
launchctl start ~/Library/LaunchAgents/com.openclaw.gateway.plist
```

---

## Configure OpenRouter (All 4 LLMs in One)

Edit the config file at `~/.openclaw/openclaw.json`:

```json
{
  "models": {
    "primary": "openrouter/anthropic/claude-sonnet-4-6",
    "subagent": "openrouter/deepseek/deepseek-v3.2",
    "heartbeat": "openrouter/google/gemini-2.5-pro"
  },
  "providers": {
    "openrouter": {
      "apiKey": "YOUR_OPENROUTER_API_KEY"
    }
  }
}
```

Get your OpenRouter API key at [openrouter.ai](https://openrouter.ai) — one key for all 4 models.

**Model routing strategy:**
- `primary` → Claude Sonnet (complex reasoning, final reports)
- `subagent` → DeepSeek V3.2 (bulk research, crawling — cheapest)
- `heartbeat` → Gemini 2.5 Pro (lightweight checks)
- GPT-4o → call explicitly when you need built-in web search

---

## Telegram Channel Setup (Recommended First Channel)

Telegram is the easiest channel to connect — takes ~5 minutes.

1. Open Telegram, search for **@BotFather**
2. Send `/newbot` — follow prompts, get a **bot token**
3. During OpenClaw onboarding (or after via `openclaw channel add telegram`):
   - Paste your bot token
   - OpenClaw connects automatically
4. Start a chat with your new bot in Telegram
5. Send a message — OpenClaw responds

---

## Install Crawl4AI Skill (Web Research)

After onboarding, install the web crawling skill:

```bash
openclaw skill install crawl4ai
```

This enables agents to crawl and scrape web pages for research.

---

## Security Hardening

`~/.openclaw/` is a high-privilege directory — treat it like a password vault.

- Never share your auth token
- Set `allowlist` to your Telegram user ID only (prevents strangers from accessing your agent)
- Keep `~/.openclaw/openclaw.json` permissions restricted:
  ```bash
  chmod 600 ~/.openclaw/openclaw.json
  ```
- Disable DMs from unknown users in channel settings

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

## Setup Order (Step by Step)

- [ ] 1. Install Homebrew
- [ ] 2. Install Node.js 24 via Homebrew
- [ ] 3. Install pnpm
- [ ] 4. Install OpenClaw via pnpm
- [ ] 5. Run `openclaw onboard --install-daemon`
- [ ] 6. Create Telegram bot via BotFather
- [ ] 7. Connect Telegram channel during onboarding
- [ ] 8. Get OpenRouter API key
- [ ] 9. Configure `~/.openclaw/openclaw.json` with OpenRouter
- [ ] 10. Install Crawl4AI skill
- [ ] 11. Security hardening
- [ ] 12. Test — send first message via Telegram

---

## Sources

- [OpenClaw Official Docs — Install](https://docs.openclaw.ai/install)
- [OpenClaw macOS Install Guide — AlexHost](https://alexhost.com/faq/how-to-install-openclaw-on-macos/)
- [How to Install OpenClaw on Mac — Medium/Zilliz](https://medium.com/@zilliz_learn/how-to-install-and-run-openclaw-previously-clawdbot-moltbot-on-mac-9cb6adb64eef)
- [OpenClaw Install Guide 2026 — Boilerplate Hub](https://boilerplatehub.com/blog/openclaw-install)
- [How to Run OpenClaw: Daemon & TUI — Dextra Labs](https://dextralabs.com/blog/how-to-run-openclaw/)
- [OpenRouter + OpenClaw Integration](https://openrouter.ai/docs/guides/guides/openclaw-integration)
