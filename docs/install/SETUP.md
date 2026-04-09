# OpenClaw Setup — How We Actually Did It

**Date:** 2026-03-30 to 2026-04-01  
**Hardware:** Mac Mini M4, 16GB RAM, 512GB SSD  
**Status:** Running ✅

---

## 1. Install OpenClaw

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Installs Homebrew, Node.js 24, and OpenClaw CLI automatically.

```bash
openclaw onboard --install-daemon
```

Wizard choices:
- LLM provider: **OpenRouter** (paste `OPENROUTER_API_KEY`)
- Channel: **Telegram** (paste bot token from @BotFather)
- Workspace: default (`~/.openclaw/workspace`)

---

## 2. Config (`~/.openclaw/openclaw.json`)

Key settings we added/changed:

```json
{
  "env": {
    "OPENROUTER_API_KEY": "...",
    "TELEGRAM_BOT_TOKEN": "...",
    "EXA_API_KEY": "...",
    "FIRECRAWL_API_KEY": "...",
    "APPLE_ID": "thefightingbee@gmail.com",
    "APPLE_APP_PASSWORD": "xxxx-xxxx-xxxx-xxxx"
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "openrouter/deepseek/deepseek-chat"
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "dmPolicy": "allowlist",
      "allowFrom": ["6495581955"]
    }
  }
}
```

**Default model:** DeepSeek V3 via OpenRouter (~20x cheaper than Sonnet, same quality for daily tasks)

---

## 3. Gmail + Google Calendar (gog skill)

```bash
brew install steipete/tap/gogcli
openclaw skills install gog
```

OAuth setup:
1. Google Cloud Console → new project → enable Gmail API + Google Calendar API
2. Create OAuth credentials → Desktop app → download `client_secret.json`
3. Run:

```bash
gog auth credentials ~/path/to/client_secret.json
gog auth add thefightingbee@gmail.com --services gmail,calendar
gog auth add gene@smtown.com --services gmail,calendar
```

Verify: `gog auth list`

---

## 4. iCloud Calendar (icalendar-sync)

**Why not icloud-caldav?** The `icloud-caldav` skill's `caldav.py` has no timeout — it hangs indefinitely on the "Me" calendar. Replaced with `icalendar-sync` which uses macOS Calendar.app native bridge (no network call, instant).

```bash
# Install
openclaw skills install icalendar-sync
cp -r ~/.openclaw/skills/icalendar-sync ~/.openclaw/workspace/skills/
cd ~/.openclaw/workspace/skills/icalendar-sync
pip3 install --user .
```

Test:
```bash
PYTHONPATH=~/.openclaw/workspace/skills/icalendar-sync/src \
ICLOUD_USERNAME="$APPLE_ID" \
ICLOUD_APP_PASSWORD="$APPLE_APP_PASSWORD" \
/Users/genehan/Library/Python/3.9/bin/icalendar-sync list --provider auto
```

Calendars available: `Me`, `우리` (shared family), `Reminders`

---

## 5. Cron Jobs

Both jobs in `~/.openclaw/cron/jobs.json`.

### Daily Morning Briefing
- **Schedule:** `0 9 * * *` (9 AM KST)
- **Model:** `openrouter/deepseek/deepseek-chat`
- **Cost:** ~$0.006/run
- **Delivery:** Telegram

Checks:
- Personal Gmail (`category:personal newer_than:1d --max 50`)
- Work Gmail (`newer_than:1d --max 50`)
- iCloud calendar: Me + 우리 (via icalendar-sync)
- Work Google Calendar (via gog)

### Weekly Review
- **Schedule:** `0 22 * * 0` (Sunday 10 PM KST)
- **Model:** `openrouter/deepseek/deepseek-chat`
- **Delivery:** Telegram

Checks:
- iCloud calendar: Me + 우리, 14-day window (this weekend + next week)
- Work Google Calendar: next week

---

## 6. Gateway Auto-Restart

Fixed missing `KeepAlive` so gateway restarts if it crashes:

```bash
openclaw doctor --repair
```

---

## 7. What's Running Now

```bash
openclaw gateway status   # check if running
openclaw cron list        # see scheduled jobs
openclaw cron runs --id <id>  # check recent run results
```

Dashboard: `http://127.0.0.1:18789/`
