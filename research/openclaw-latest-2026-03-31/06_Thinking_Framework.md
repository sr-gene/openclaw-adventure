# 06 — Thinking Framework

Decision trees, selection guides, and quick-reference tables for operating OpenClaw on Mac Mini M4 for financial research.

> **Source notes:** Synthesized from workspace research notes dated 2026-03-30.

<!-- IMAGE: Decision tree flowchart for model selection and pipeline routing -->

---

## Decision Tree 1 — Installation Method

```
Is this a personal Mac (not a server)?
├── YES → Are you comfortable with Terminal?
│         ├── YES → Use: pnpm add -g openclaw@latest  (most control)
│         └── NO  → Use: curl -fsSL https://openclaw.ai/install.sh | bash  (one-liner)
└── NO  → Is this a shared/server environment?
          ├── YES → Use: Docker
          └── NO  → Build from source (developer use)
```

**For Mac Mini M4 personal setup: one-liner or pnpm. Never Docker.**

---

## Decision Tree 2 — Which Model to Use

```
What is the task?
│
├── Final investment thesis / synthesis report
│   └── → Claude Sonnet 4.6  (openrouter/anthropic/claude-sonnet-4-6)
│
├── Web search with real-time data
│   └── → GPT-4o  (openrouter/openai/gpt-4o)
│
├── Structured data extraction / metrics analysis
│   └── → Gemini 2.5 Pro  (openrouter/google/gemini-2.5-pro)
│
├── Bulk web scraping / summarizing / first-pass reading
│   └── → DeepSeek V3.2  (openrouter/deepseek/deepseek-v3.2)
│
└── Scheduled heartbeat / lightweight checks
    └── → Gemini 2.5 Pro  (cheapest capable model for light tasks)
```

**Cost rule: DeepSeek handles 90% of volume. Claude Sonnet handles final synthesis only.**

---

## Decision Tree 3 — Skill Selection for Web Research

```
Need to scrape a website for research data?
│
├── Is the site behind JS rendering / requires real browser?
│   ├── YES → Use: Firecrawl skill  (clawhub install firecrawl)
│   │          Remote browser, no local RAM, handles 403s
│   └── NO  → Firecrawl still works fine; use it by default
│
└── Want local crawling with full control?
    └── Use: crawl4ai (check for official OpenClaw skill — may be manual setup)
        Note: Crawl4AI has no confirmed official ClawHub skill as of 2026-03-30 [unverified]
```

**Recommendation: Use Firecrawl as the default web research skill.**

---

## Decision Tree 4 — Telegram Access Problem

```
Bot not responding to my message?
│
├── Is the daemon running?
│   ├── NO  → openclaw gateway start
│   └── YES → Continue...
│
├── Is my numeric ID in allowFrom?
│   ├── NO  → Add to openclaw.json, restart daemon
│   └── YES → Continue...
│
├── Is the bot token correct?
│   ├── NO  → Regenerate via @BotFather /token, update config
│   └── YES → Continue...
│
└── Run: openclaw logs --follow
    → Check for errors, look for your message in the log stream
```

---

## Decision Tree 5 — Daemon Issues on macOS

```
Daemon behaving unexpectedly?
│
├── After running "openclaw gateway restart" → daemon broken?
│   └── Known KeepAlive bug. Always use:
│       openclaw gateway stop && openclaw gateway start
│
├── Daemon stops working after Mac sleep/wake?
│   └── Apply conditional KeepAlive plist fix (see 02_Configuration.md)
│
├── Daemon not found by launchctl?
│   └── launchctl load ~/Library/LaunchAgents/com.openclaw.gateway.plist
│
└── Persistent crashes?
    └── openclaw doctor --fix
```

---

## Model Quick-Reference Table

| Model | Cost (in/out per 1M) | Speed | Best Task | Avoid For |
|-------|---------------------|-------|-----------|-----------|
| Claude Sonnet 4.6 | $3 / $15 | Medium | Final synthesis, complex reasoning | Bulk scraping (expensive) |
| GPT-4o | $2.50 / $10 | Fast | Web search, tool use | Pure text analysis (overpaying) |
| Gemini 2.5 Pro | $1.25 / $5 | Fast | Structured extraction, analysis | Final thesis (weaker reasoning) |
| DeepSeek V3.2 | $0.28 / $0.42 | Fast | Bulk summarizing, scraping | Final synthesis (weaker reasoning) |

[All prices estimated based on published rates as of Q1 2026 — verify at openrouter.ai]

---

## Command Quick-Reference Table

| Task | Command |
|------|---------|
| Check daemon status | `openclaw gateway status` |
| Start daemon | `openclaw gateway start` |
| Stop daemon | `openclaw gateway stop` |
| Restart daemon (safe) | `openclaw gateway stop && openclaw gateway start` |
| Watch live logs | `openclaw logs --follow` |
| Open dashboard | `openclaw dashboard` |
| Run health check | `openclaw doctor` |
| Fix common issues | `openclaw doctor --fix` |
| Run security audit | `openclaw security audit` |
| Fix security issues | `openclaw security audit --fix` |
| Install a skill | `clawhub install <skill-name>` |
| List installed skills | `clawhub list` [unverified] |
| View skill details | `clawhub info <skill-name>` [unverified] |
| Add Telegram channel | `openclaw channel add telegram` |
| Re-run onboarding | `openclaw onboard` |

---

## Monthly Cost Estimator

| Usage Level | Tasks/Day | Est. Monthly Cost |
|-------------|-----------|-----------------|
| Light | 5–10 | ~$55 |
| Medium | 20–30 | ~$161 |
| Heavy | 50+ | ~$320 |

[Estimates from workspace research notes — assumes DeepSeek for 90% of tasks, Claude for final synthesis]

**Cost reduction tips:**
1. Route all scraping/summarizing to DeepSeek (saves ~80% vs using Claude for everything)
2. Cache scraped data in workspace files — don't re-scrape the same source twice
3. Set token limits per sub-agent to prevent runaway costs
4. Review `openclaw logs` weekly to spot unexpected model usage

---

## Pipeline Selection Guide

| Research Request Type | Recommended Pipeline | Primary Model |
|-----------------------|---------------------|---------------|
| Single company deep-dive | Sequential 4-stage | Claude orchestrator + DeepSeek fetch |
| Market sector scan | Parallel fan-out (3–5 sources) | DeepSeek all fetchers, Claude synthesis |
| News briefing (daily) | Simple 2-stage (fetch → summarize) | DeepSeek only (cheap) |
| Competitor comparison | Parallel fan-out + merged analysis | DeepSeek fetch, Gemini analyze, Claude thesis |
| Quick fact check | Single-agent direct | Gemini 2.5 Pro (fast, cheap) |

---

## Setup Order Reference

The recommended sequence for a fresh Mac Mini M4 setup:

```
[ ] 1. Install Homebrew
[ ] 2. Install Node.js 24 via Homebrew
[ ] 3. Install pnpm
[ ] 4. Install OpenClaw: pnpm add -g openclaw@latest
[ ] 5. Run: openclaw onboard --install-daemon
      (provider: OpenRouter, channel: Telegram)
[ ] 6. Security hardening:
      chmod 700 ~/.openclaw
      chmod 600 ~/.openclaw/openclaw.json
      openclaw security audit --fix
[ ] 7. Apply KeepAlive plist fix (02_Configuration.md)
[ ] 8. Create Telegram bot via @BotFather
[ ] 9. Add numeric Telegram ID to allowFrom config
[ ] 10. clawhub install firecrawl
[ ] 11. Configure workspace files (SOUL.md, USER.md, AGENTS.md, HEARTBEAT.md)
[ ] 12. Test: send first message via Telegram
[ ] 13. Version-control ~/.openclaw/workspace/ in private Git repo
```

---

## Sources

- Workspace notes: `research/setup-guide-2026-03-30.md`
- Workspace notes: `research/getting-started-openclaw-2026-03-30.md`
- Workspace notes: `research/mac-mini-specs-2026-03-30.md`
