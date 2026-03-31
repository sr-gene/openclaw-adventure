# Gmail & Google Calendar Integration with OpenClaw

Research date: 2026-03-30
OpenClaw version target: 2026.3.x

---

## Overview

OpenClaw connects to Gmail and Google Calendar through **gog** — a first-party Google Workspace CLI skill that ships with OpenClaw's skill registry. It uses standard OAuth 2.0, so no passwords are stored and access can be revoked at any time from Google Account settings. All processing happens locally.

---

## 1. Installing Skills / Plugins in OpenClaw 2026.x

OpenClaw has two parallel interfaces for skill management as of 2026: native `openclaw` commands and the separate `clawhub` CLI. Either works; native commands are preferred because they persist ClawHub source metadata for seamless future updates.

### Native OpenClaw commands (recommended)

```bash
# Search the registry
openclaw skills search "gmail"
openclaw skills search "calendar"

# Install a skill
openclaw skills install <skill-slug>

# Update all installed skills
openclaw skills update --all

# Install a plugin (code extensions, not prompt-based skills)
openclaw plugins install clawhub:<package-name>
openclaw plugins update --all
```

After installing any skill, start a new OpenClaw session to activate it.

### ClawHub CLI (alternative)

```bash
# Install the ClawHub CLI globally
npm i -g clawhub
# or
pnpm add -g clawhub

# Search, install, update
clawhub search "gmail"
clawhub install <slug>
clawhub install <slug> --version <version> --force
clawhub update --all
clawhub list
```

**Security note:** As of early 2026, ~3% of ClawHub's 13,000+ skills were found to be malicious (prompt injection + malware). Stick to official openclaw/* namespace skills and well-reviewed community skills with many installs.

---

## 2. Connecting Gmail to OpenClaw

Gmail is handled by the **gog** skill (Google Operations Gateway). It wraps the Gmail API with OAuth 2.0 and exposes email operations to the agent.

### Step 1: Install gog

The gog binary is installed via Homebrew (separate from the OpenClaw skill registration):

```bash
brew install steipete/tap/gogcli
```

Also register it as an OpenClaw skill so the agent can invoke it:

```bash
openclaw skills install gog
# or via clawhub:
clawhub install gog
```

### Step 2: Create Google Cloud OAuth credentials

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project (e.g., "OpenClaw Assistant") or select an existing one
3. Enable the **Gmail API** (APIs & Services > Library > search "Gmail API" > Enable)
4. Go to APIs & Services > Credentials > Create Credentials > OAuth client ID
5. Application type: **Desktop app**
6. Download the `client_secret.json` file

### Step 3: Authenticate gog with Gmail

```bash
# Point gog at your credentials file
gog auth credentials /path/to/client_secret.json

# Authorize your Gmail account (opens browser for OAuth consent)
gog auth add you@gmail.com --services gmail,calendar,drive,contacts,docs,sheets

# Verify setup
gog auth list
```

Set `GOG_ACCOUNT=you@gmail.com` in your shell profile to avoid passing `--account` on every command.

### Step 4: Test Gmail access

```bash
gog gmail search 'newer_than:7d' --max 10
```

### Key Gmail operations available to the agent

- Search messages: `gog gmail search '<query>'`
- Send (plain text or HTML): `gog gmail send`
- Create drafts, reply to threads
- Read/write labels

---

## 3. Connecting Google Calendar to OpenClaw

Calendar access uses the same **gog** skill and the same Google Cloud project. If you already did the Gmail setup above, the main additions are: enabling the Calendar API and including `calendar` in the `gog auth add` services list.

### Step 1: Enable Calendar API in Google Cloud Console

1. In the same Google Cloud project, go to APIs & Services > Library
2. Search "Google Calendar API" and click **Enable**
   - Note: Even with gog installed, Google blocks calendar access until this is explicitly enabled.

### Step 2: Re-authorize if you previously only granted Gmail

If you ran `gog auth add` with only `gmail`, refresh permissions:

```bash
gog auth add you@gmail.com --services gmail,calendar,drive,contacts,docs,sheets
```

(The full services list is shown above — include everything you want upfront to avoid re-doing OAuth.)

### Step 3: Test Calendar access

```bash
gog calendar today          # Today's events
gog calendar list <calId> --from <ISO date> --to <ISO date>
```

### Creating calendar events

```bash
gog calendar create <calendarId> \
  --summary "Meeting title" \
  --from 2026-04-01T09:00:00 \
  --to 2026-04-01T10:00:00
```

Calendar IDs for "primary" calendar: use `primary` or your full Gmail address.

### Two integration approaches

| Method | Pros | Cons |
|--------|------|------|
| **gog + OAuth** (recommended) | Read/write, real-time, structured | Requires Google Cloud project setup |
| **ICS feed** | Simple, no Cloud project needed | Read-only, delayed by sync interval |

---

## 4. Scheduled Reports: Cron and Heartbeat

OpenClaw has two scheduling mechanisms. Use **cron** for precise timed reports (daily briefing, weekly review). Use **heartbeat** for ambient background monitoring.

### Cron vs. Heartbeat

| Feature | Cron | Heartbeat |
|---------|------|-----------|
| Trigger | Exact schedule (cron expression or interval) | Every N minutes (default: 30m) |
| Session | Isolated (own context) or main | Main session only |
| Context | No prior context in isolated mode | Full conversation context |
| Best for | Reports, briefings, weekly reviews | Passive monitoring, surfacing urgent items |
| Storage | `~/.openclaw/cron/jobs.json` | `HEARTBEAT.md` in project |

### Daily Morning Briefing (7 AM)

```bash
openclaw cron add \
  --name "Morning briefing" \
  --cron "0 7 * * *" \
  --tz "America/New_York" \
  --session isolated \
  --message "Generate today's briefing: check unread emails for anything urgent, pull today's calendar events, summarize weather, and give a brief news digest. Format as a clean daily brief." \
  --model opus \
  --announce \
  --channel whatsapp \
  --to "+15551234567"
```

For weekdays only, change `--cron "0 7 * * *"` to `--cron "0 7 * * 1-5"`.

### Weekly Sunday Report (9 AM)

```bash
openclaw cron add \
  --name "Weekly review" \
  --cron "0 9 * * 0" \
  --tz "America/New_York" \
  --session isolated \
  --message "Generate a weekly review: summarize emails from the past 7 days, review calendar events from the past week and upcoming week, flag incomplete tasks, and highlight key themes or priorities." \
  --model opus \
  --announce \
  --channel whatsapp \
  --to "+15551234567"
```

Note: `0 9 * * 0` = Sunday at 9 AM. Use `* * 1` for Monday instead.

### One-shot reminder

```bash
openclaw cron add \
  --name "One-time reminder" \
  --at "2026-04-01T09:00:00" \
  --tz "America/New_York" \
  --message "Remind me to review the Q1 report." \
  --announce \
  --channel telegram \
  --to "YOUR_CHAT_ID"
```

One-shot jobs auto-delete after successful execution.

### Heartbeat configuration

Heartbeats are configured in `openclaw.json`:

```json
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "target": "last",
        "activeHours": { "start": "08:00", "end": "22:00" }
      }
    }
  }
}
```

The agent's behavior during heartbeats is controlled by `HEARTBEAT.md` in your project:

```md
# Heartbeat checklist
- Check email for urgent messages; surface anything requiring same-day action
- Review calendar for events in the next 2 hours and send a reminder if needed
- If a background task finished, summarize results
- If idle for 8+ hours, send a brief check-in
- Do not nag — limit proactive messages to 3 per day maximum
- Skip reporting unchanged information
```

### Cron expression cheat sheet

| Expression | Meaning |
|------------|---------|
| `0 7 * * *` | Daily at 7:00 AM |
| `0 7 * * 1-5` | Weekdays at 7:00 AM |
| `0 9 * * 0` | Sundays at 9:00 AM |
| `0 9 * * 1` | Mondays at 9:00 AM |
| `30 8 * * 1-5` | Weekdays at 8:30 AM |
| `0 */6 * * *` | Every 6 hours |
| `0 5 * * *` | Daily at 5:00 AM |

Cron jobs persist in `~/.openclaw/cron/jobs.json` and survive restarts. They include automatic retry with exponential backoff and per-job execution logging.

---

## 5. Official Docs and Community Guides

### Official

- [OpenClaw GitHub](https://github.com/openclaw/openclaw) — main repo, includes `skills/gog/SKILL.md`
- [ClawHub docs](https://docs.openclaw.ai/tools/clawhub) — skill installation reference
- [Cron vs Heartbeat docs](https://docs.openclaw.ai/automation/cron-vs-heartbeat) — official scheduling guide
- [OpenClaw Google Workspace integration](https://www.getopenclaw.ai/en/integrations/google-workspace)
- [OpenClaw Cron Jobs docs](https://openclaw-ai.com/en/docs/automation/cron-jobs)

### Community guides

- [How to Connect OpenClaw to Google Calendar (Medium, Feb 2026)](https://satyatechgeek.medium.com/how-to-connect-openclaw-to-google-calendar-the-complete-guide-e996ffb93e31)
- [OpenClaw Post-Onboarding: Skills, Apple Integrations, and Google Calendar (Towards AI, Feb 2026)](https://pub.towardsai.net/openclaw-post-onboarding-skills-apple-integrations-and-google-calendar-the-secure-way-4b4b4e49dfa8)
- [How I Automated a Daily Intelligence Briefing with OpenClaw (Jose Casanova)](https://www.josecasanova.com/blog/openclaw-daily-intel-report)
- [OpenClaw Cron Jobs: Automate Your AI Agent's Daily Tasks (DEV Community)](https://dev.to/hex_agent/openclaw-cron-jobs-automate-your-ai-agents-daily-tasks-4dpi)
- [OpenClaw HEARTBEAT, SOUL, and Memory Files (Blink Blog, 2026)](https://blink.new/blog/openclaw-heartbeat-soul-memory-configuration-guide-2026)
- [OpenClaw ClawHub Setup Guide (Fast.io)](https://fast.io/resources/openclaw-clawhub-setup/)
- [OpenClaw Skills: How to Install from ClawHub Safely in 2026 (Blink Blog)](https://blink.new/blog/openclaw-clawhub-skills-safe-install-guide-2026)
- [Cron Jobs Daily Setup Guide (OpenClawReady)](https://openclawready.com/blog/openclaw-cron-jobs-daily-automation/)
- [Automation — Cron Jobs, Webhooks & Heartbeat (Learn OpenClaw)](https://learnopenclaw.com/core-concepts/automation)

### Alternative: Maton-based Gmail skill (no Google Cloud setup required)

There is a community skill (`byungkyu/gmail`) that routes Gmail through [Maton](https://maton.ai) — a managed OAuth gateway. This avoids creating a Google Cloud project but adds a third-party service dependency:

```bash
export MATON_API_KEY="YOUR_API_KEY"
```

Connections managed at [ctrl.maton.ai](https://ctrl.maton.ai). Rate limit: 10 req/sec. This is simpler but less private than the direct gog approach.

---

## Recommended Setup Order

1. Install gog: `brew install steipete/tap/gogcli`
2. Create Google Cloud project, enable Gmail API + Calendar API
3. Download `client_secret.json`
4. `gog auth credentials /path/to/client_secret.json`
5. `gog auth add you@gmail.com --services gmail,calendar,drive,contacts,docs,sheets`
6. `gog auth list` to verify
7. `openclaw skills install gog` to register with OpenClaw
8. Test: `gog calendar today` and `gog gmail search 'newer_than:1d'`
9. Add morning briefing cron job
10. Add weekly review cron job
11. Configure `HEARTBEAT.md` for ambient monitoring
