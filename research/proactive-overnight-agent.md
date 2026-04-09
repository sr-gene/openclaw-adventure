# UC-6: Proactive Overnight Agent (HEARTBEAT.md)

Research date: 2026-03-31
OpenClaw version target: 2026.3.x

---

## 1. What This Is

A 24/7 proactive agent that monitors things while you're asleep, at work, or away — and only messages you on Telegram when something actually needs attention. No spam, no polling. This is **uniquely OpenClaw** — no other app combines always-on monitoring with multi-tool action and intelligent alert suppression.

**Key concept: HEARTBEAT.md** — a checklist file that tells OpenClaw what to check on a recurring interval (default: every 30 minutes). If nothing needs attention, it stays silent (`HEARTBEAT_OK`). If something triggers, it alerts you via Telegram.

---

## 2. What It Monitors

### Email (Gmail — via `gog` skill)
- Urgent unread emails from VIP senders (boss, wife, family)
- Emails requiring same-day response
- Unexpected bills or payment confirmations over threshold

### Calendar (iCloud — via `icloud-caldav` + `apple-calendar` skills)
- Upcoming events in the next 12 hours (prep reminders)
- Calendar conflicts (double-bookings)
- New events added to shared family calendar by wife

### Financial (future — after UC-3 & UC-4 setup)
- Portfolio alerts: stock moves > 3% in either direction
- Breaking news affecting held positions
- Unusual spending alerts from finance.db

### News & World Events
- Breaking news on tracked topics (AI, geopolitics, markets)
- Only alerts for high-impact events, not routine news

---

## 3. How HEARTBEAT.md Works

### File location
```
~/openclaw/workspace/HEARTBEAT.md
```

### Format
Standard markdown checkboxes. Active items = `- [ ]`. Completed one-time items = `- [x]`.

### Example HEARTBEAT.md for our setup

```md
# Heartbeat Checklist

## Email Monitoring
- [ ] Check Gmail for unread emails from VIP list (wife, family, boss). If urgent, alert immediately via Telegram with subject + sender + one-line summary.
- [ ] Check for emails with attachments over 10MB or payment/invoice keywords. Flag if found.

## Calendar Awareness
- [ ] Check iCloud calendar for events in the next 6 hours. If there's an event within 2 hours that hasn't been reminded yet, send a prep reminder via Telegram.
- [ ] Check if wife added any new events to the shared family calendar in the last hour. If so, notify via Telegram.

## Financial Monitoring (enable after UC-3/UC-4)
# - [ ] Check watchlist stocks (AAPL, GOOGL, NVDA, MSFT) for moves > 3%. Alert if triggered.
# - [ ] Check finance.db for any new transactions over 500,000원. Alert with details.

## System Health
- [ ] Verify OpenClaw gateway is running and responsive. If any skill errors occurred in the last interval, log them.
```

### Alert behavior
- If nothing needs attention → agent replies `HEARTBEAT_OK` → **suppressed, you see nothing**
- If something triggers → agent sends alert content to Telegram → **you get notified**
- Replies ≤ 300 chars with `HEARTBEAT_OK` are treated as acknowledgments and hidden

---

## 4. Configuration

Add to `~/.openclaw/openclaw.json`:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",                    // check every 30 minutes
        target: "last",                  // send alerts to last active channel (Telegram)
        lightContext: true,              // only load HEARTBEAT.md, not full context
        isolatedSession: true,           // fresh session each run (saves ~100K tokens)
        includeReasoning: false,         // don't include chain-of-thought in alerts
        activeHours: {
          start: "00:00",               // run 24/7 (adjust if you want quiet hours)
          end: "23:59",
          timezone: "Asia/Seoul"
        }
      }
    }
  }
}
```

### Cost-saving settings explained

| Setting | What it does | Why |
|---------|-------------|-----|
| `lightContext: true` | Only loads HEARTBEAT.md into context | Reduces from ~100K tokens to ~2-5K per run |
| `isolatedSession: true` | Fresh session, no conversation history | Avoids sending full chat history every 30 min |
| `includeReasoning: false` | Suppresses chain-of-thought | Cleaner alerts, fewer tokens |

### Active hours options

```json5
// Option A: 24/7 monitoring (recommended for email/calendar)
activeHours: { start: "00:00", end: "23:59", timezone: "Asia/Seoul" }

// Option B: Waking hours only (save cost)
activeHours: { start: "07:00", end: "23:00", timezone: "Asia/Seoul" }

// Option C: Overnight only (complement to daytime cron jobs)
activeHours: { start: "22:00", end: "07:00", timezone: "Asia/Seoul" }
```

---

## 5. Model Selection

Heartbeat runs frequently, so cost matters:

| Model | Cost per heartbeat | Monthly (48 runs/day) | Best for |
|-------|-------------------|----------------------|----------|
| **Haiku** | ~$0.001 | ~$1.50/month | Simple checks (email count, calendar scan) |
| **DeepSeek V3.2** | ~$0.0005 | ~$0.75/month | Bulk monitoring, news scanning |
| **Sonnet** | ~$0.005 | ~$7.50/month | Complex analysis (financial, multi-step) |

**Decision:** Use **DeepSeek V3.2** for the heartbeat loop — cheapest option at ~$0.75/month for 24/7 monitoring. It handles simple "check and alert" patterns well. Reserve Sonnet for actual alert content generation when something triggers.

---

## 6. Alert Examples

### What you'd see on Telegram

**Urgent email alert:**
```
🔴 Urgent Email
From: Wife
Subject: 내일 병원 예약 변경됨
Received: 11:43 PM
Summary: Tomorrow's hospital appointment moved from 2 PM to 10 AM.
→ Your calendar still shows 2 PM. Want me to update it?
```

**Calendar conflict:**
```
⚠️ Calendar Conflict Detected
Tomorrow (April 1):
  10:00 AM - 병원 예약 (shared calendar, added by wife)
  10:00 AM - Team standup (work calendar)
These overlap. Want me to reschedule the standup?
```

**Financial alert (future):**
```
📊 Stock Alert
NVDA dropped 4.2% after hours ($892 → $854)
Reason: Reuters reports new export restrictions to China
Your exposure: 15% of portfolio
→ Want a full impact analysis?
```

**Nothing happening (you never see this):**
```
HEARTBEAT_OK
```

---

## 7. Implementation Roadmap

| Step | Task | Effort |
|------|------|--------|
| 1 | Create HEARTBEAT.md with email + calendar checks only | 15 min |
| 2 | Configure heartbeat settings in openclaw.json | 10 min |
| 3 | Test with 30-min interval, verify alerts arrive on Telegram | 30 min |
| 4 | Tune VIP sender list, alert thresholds | 15 min |
| 5 | Add financial monitoring after UC-3/UC-4 are live | Later |
| 6 | Add news monitoring after UC-1/UC-2 are live | Later |

**Total initial setup: ~1 hour**

---

## 8. Cost Estimate

| Scenario | Interval | Model | Monthly cost |
|----------|----------|-------|-------------|
| **Light** (waking hours, 16h/day) | 30 min | DeepSeek V3.2 | ~$0.50 |
| **Standard** (24/7) | 30 min | DeepSeek V3.2 | ~$0.75 |
| **Heavy** (24/7 + complex analysis) | 15 min | Sonnet | ~$15.00 |

**Chosen config:** 24/7, 30-min interval, DeepSeek V3.2 = **~$0.75/month**

---

## 9. What Makes This Uniquely OpenClaw

No single app or service does all of this:

| Capability | Regular apps | OpenClaw |
|-----------|-------------|---------|
| Email alerts | Gmail notifications (noisy, no intelligence) | Smart triage — only alerts for truly urgent items |
| Calendar reminders | Apple Calendar (static, dumb) | Context-aware — knows your email context, can detect conflicts across calendars |
| Stock alerts | Yahoo Finance (threshold only) | Cross-references news + portfolio + your risk profile |
| Action on alerts | None — you have to act | Can offer to reschedule, draft replies, update calendar |
| Multi-source correlation | Impossible | "Wife emailed about dinner change + calendar updated + restaurant booking needs canceling" — all connected |

The killer feature isn't any individual check — it's the **agent connecting dots across email, calendar, finances, and news** and taking action, not just notifying.

---

## 10. Open Questions

| Item | Decision needed |
|------|----------------|
| Quiet hours | Do you want alerts suppressed during sleep (e.g., 12 AM - 7 AM)? Or 24/7? |
| VIP sender list | Who should trigger immediate alerts? Wife, family, boss? |
| Alert threshold | What email keywords trigger alerts? (e.g., "urgent", "ASAP", "결제", "예약") |
| Financial alerts | Enable after UC-3/UC-4, or start with basic stock watchlist now? |

---

## Sources

- [OpenClaw Heartbeat Docs](https://docs.openclaw.ai/gateway/heartbeat)
- [OpenClaw Heartbeat GitHub Source](https://github.com/openclaw/openclaw/blob/main/docs/gateway/heartbeat.md)
- [Proactive Agent Guide — Sam's Playbook](https://openclawsetup.info/en/blog/openclaw-heartbeat-proactive-agents)
- [OpenClaw HEARTBEAT, SOUL, and Memory Config Guide](https://blink.new/blog/openclaw-heartbeat-soul-memory-configuration-guide-2026)
- [OpenClaw Showcase](https://openclaw.ai/showcase)
