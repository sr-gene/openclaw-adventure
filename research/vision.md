# OpenClaw Vision & Goals

## Overview

This document captures the full vision for running OpenClaw as a personal AI agent gateway on a Mac Mini M4. The goal is to automate financial research, personal intelligence, and household management — all delivered to Telegram.

**Hardware:** Mac Mini M4, 16GB RAM, 512GB SSD
**Inference:** Cloud APIs only (no local models)
**Primary delivery channel:** Telegram

---

## Setup Order

| Priority | Use Case | Status |
|----------|----------|--------|
| 1st | UC-5: Gmail + Calendar Daily/Weekly Reports | Done |
| 2nd | UC-3: Household Account Book | Next up |
| 3rd | UC-1: Twitter/X Intelligence Reports | — |
| 4th | UC-2: News Crawler + Portfolio Impact Reports | — |
| 5th | UC-4: Financial Research & Investment Strategy | — |
| 6th | UC-6: Proactive Overnight Agent (HEARTBEAT.md) | Research done |
| 7th | UC-7: Fashion Market Research (TIME 타임 MD) | Research done |

---

## Use Cases

### UC-1: Twitter/X Intelligence Reports

Monitor Twitter/X for trending discussions and sentiment on key topics.

**Topics to track:**
- AI tools: OpenClaw, Claude Code, Cursor, Windsurf
- AI companies: Anthropic, OpenAI, Google DeepMind, Meta AI
- General AI narrative and hype cycles

**What the agent does:**
- Crawls recent posts by topic/hashtag
- Identifies what's being discussed, what's hyping, and dominant opinions
- Flags contrarian or bearish voices against the mainstream narrative
- Produces a structured sentiment report

**Output:** Weekly Twitter/X intelligence digest via Telegram

**Implementation note:** Twitter/X API v2 free tier is severely rate-limited (500k tweets/month cap, no full-archive search). Recommended approach: **Apify Twitter Scraper** skill or a RapidAPI Twitter endpoint. Budget ~$5–20/month for Apify depending on volume.

---

### UC-2: News Crawler + Portfolio Impact Reports

Monitor news on specific topics and tie it directly to portfolio holdings.

**Topics to track:**
- Geopolitical events (e.g., Middle East conflicts, Taiwan, US-China trade)
- Macro economics (Fed policy, inflation, jobs data)
- Sector-specific news for holdings

**Sources:**
- New York Times (paid subscription — auth via NYT API key or cookie-based session)
- Reuters, Bloomberg, Financial Times, Seeking Alpha
- Google News aggregation via Firecrawl

**What the agent does:**
- Crawls top stories on tracked topics
- Maps each story to relevant portfolio positions
- Produces a "how does today's news affect your holdings" report with bull/bear framing

**Output:** On-demand report (trigger via Telegram) + daily headline digest

**Implementation note:** NYT has a developer API (limited free tier). For full article access with a paid subscription, cookie-based Firecrawl scraping is the pragmatic approach. NYT API key covers article metadata and search.

---

### UC-3: Household Account Book

Aggregate bank data from both accounts into a unified household finance view.

**What the agent does:**
- Pulls transactions from user's bank and wife's bank
- Categorizes spending (groceries, dining, utilities, subscriptions, etc.)
- Tracks monthly income vs. expenses
- Flags unusual charges or spending spikes
- Produces a clean monthly household finance report

**Output:** Monthly household finance report via Telegram

**Important constraint:** Direct bank screen scraping violates most banks' Terms of Service and is fragile (breaks on UI changes, may trigger fraud alerts). The correct approach is a financial data aggregator:

| Option | Notes |
|--------|-------|
| **Plaid** (recommended) | Industry standard, supports most US banks, free tier available, OAuth-based |
| **Teller.io** | Developer-friendly, US banks, flat monthly fee |
| **Akahu** | NZ/AU banks |

Set up via an OpenClaw custom skill or MCP tool that wraps the Plaid API. Plaid development credentials are free; production requires an application. For personal use, development mode supports up to 100 Items (bank connections) at no cost.

---

### UC-4: Financial Research & Investment Strategy

Multi-agent financial research pipeline for both long-term and short-term plays.

**Long-term (ETF / index / sector):**
- Research ETF composition, expense ratios, historical performance, sector exposure
- Monitor rebalancing opportunities
- Track macro themes (AI infrastructure, energy transition, defense) and relevant ETFs
- Produce quarterly portfolio review reports

**Short-term (higher risk / speculative):**
- Momentum plays: scan for unusual volume, price breakouts, options activity
- Sentiment-driven trades: Twitter/news hype crossreferenced with technical signals
- Earnings plays: pre-earnings research report for held positions

**Agent pipeline:**
```
Research request (Telegram)
  │
  ▼
Claude (Orchestrator) — scope + subtopic mapping
  ├─► DeepSeek (Data Fetcher) — Firecrawl scrape, SEC filings, earnings data
  ├─► Gemini (Metrics Analyst) — quantitative analysis, ratios, comparables
  ├─► Gemini (Risk Assessor) — downside scenarios, macro risks
  └─► Claude (Thesis Writer) — synthesis → investment thesis
  │
  ▼
Claude (Orchestrator) — compiles final report
  │
  ▼
Telegram delivery
```

**Output:** On-demand investment research report with clear Buy / Hold / Avoid rating + reasoning

---

### UC-5: Gmail + Calendar Daily/Weekly Reports

Personal inbox and calendar awareness without manual checking.

**Daily briefing (7:00 AM, weekdays):**
- Unread emails: flag anything requiring same-day action
- Today's calendar events (iCloud — includes wife's shared calendars) with prep notes if relevant
- Top 3 news headlines on tracked topics
- Weather

**Weekly review (Sunday 9:00 AM):**
- Email digest: key threads from the past week
- Calendar: what happened + what's coming up next week (both personal + shared family calendar)
- Tasks/follow-ups surfaced from email threads
- Financial week-in-review (portfolio performance, key macro events)

**Integrations:**
- **Gmail:** `gog` skill (Google Operations Gateway) — OAuth 2.0 via Google Cloud project
- **iCloud Calendar:** `icloud-caldav` skill (CalDAV direct) + `apple-calendar` skill (AppleScript local reads). Uses app-specific password from appleid.apple.com. Sees all calendars including wife's shared calendars. Full read/write, real-time sync.

**Output:** Formatted daily/weekly brief delivered to Telegram

---

### UC-6: Suggested Additional Use Cases

Ideas worth building once the core use cases are running:

| Use Case | What it does |
|----------|-------------|
| **Earnings calendar tracker** | Before each earnings week, pull analyst estimates vs. prior actuals for portfolio holdings. Deliver preview report. |
| **Macro economic pulse** | Weekly digest of Fed/CPI/jobs data releases: what printed, market reaction, portfolio implications. |
| **Real estate market monitor** | Track price trends and inventory in target neighborhoods via Zillow/Redfin scrape. Useful for future home purchase planning. |
| **Apple Health weekly digest** | Export Apple Health data (steps, sleep, HRV, activity) and generate weekly personal health trend report. |
| **Utility & subscription audit** | Monthly: scan Gmail for utility bills and recurring subscription charges. Flag increases or subscriptions that haven't been used. |
| **YouTube/podcast topic digest** | Weekly summary of new content from followed channels and podcasts on AI, investing, and geopolitics. |
| **Competitor/company watch** | Track a watchlist of companies via SEC filings (EDGAR), press releases, and news. Surface anything material within 24 hours. |

---

## LLM Stack & Cost Strategy

**Rule: Let DeepSeek handle 90% of the work. Use Claude only for final synthesis.**

| Role | Model | Est. cost |
|------|-------|-----------|
| Bulk scraping, summarizing, data gathering | DeepSeek V3.2 | $0.28 in / $0.42 out per 1M |
| Analysis, metrics, risk assessment | Gemini 2.5 Pro | $1.25 in / $5 out per 1M |
| Web research with built-in search | GPT-4o | $2.50 in / $10 out per 1M |
| Final synthesis, strategy reports | Claude Sonnet 4.6 | $3 in / $15 out per 1M |

**No OpenRouter** — direct API keys per provider (avoids ~8% markup).

### Monthly cost estimate

| Usage level | Est. cost |
|-------------|-----------|
| Light (5–10 tasks/day) | ~$55/mo |
| Medium (20–30 tasks/day) | ~$161/mo |
| Heavy (50+ tasks/day) | ~$320/mo |

---

## Automation Schedule

| Time | Job | Channel |
|------|-----|---------|
| 7:00 AM daily (weekdays) | Morning briefing: email + calendar + news | Telegram |
| 9:00 AM Sunday | Weekly review | Telegram |
| Every 30 min (8 AM–10 PM) | Heartbeat: urgent email/calendar monitoring | Telegram (on trigger only) |
| Market open (weekdays) | Pre-market brief for active positions | Telegram |

---

## Open Questions for Setup Phase

| Item | Decision needed |
|------|----------------|
| Twitter/X scraping | Apify Twitter Scraper vs. RapidAPI — compare cost and rate limits |
| NYT access | NYT API key (metadata only) vs. Firecrawl with subscriber cookie |
| Bank aggregation | Plaid (free dev tier) vs. Teller.io — check which supports your specific banks |
| Portfolio data source | Manual list in config vs. brokerage API (Interactive Brokers, Schwab OAuth) |
| Wife's buy-in | Shared Plaid access requires her consent and OAuth flow |
