# Mac Mini Spec Research & Session 1 Conclusions

## What is OpenClaw?

OpenClaw is a self-hosted personal AI agent gateway ([github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)) that runs as a persistent daemon connecting LLMs to your messaging apps and local system. Key capabilities:

- Responds via WhatsApp, Telegram, iMessage, Discord, Slack, and 20+ other platforms
- Executes shell commands, manages files, automates browser tasks
- Runs scheduled tasks (cron), handles webhooks, controls a Chrome/Chromium instance
- Extends via "skills" from ClawHub (public skill registry)

**Software requirements:** Node.js 22+, macOS/Linux/WSL2.

---

## Use Case

Financial research and strategy automation using **multiple concurrent agents** doing web crawling, data gathering, and analysis — all routed to cloud LLMs. No local models. Results delivered via Telegram.

---

## Hardware Decision: Mac Mini M4, 16GB RAM, 512GB SSD

**PURCHASED**

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

---

## LLM Stack (Cloud APIs Only)

Accessed via **OpenRouter** (single API key, native OpenClaw integration):

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
| OpenRouter | Unified API access to all 4 LLMs |
| Crawl4AI | Local web crawling, LLM-ready markdown output |
| Telegram | First messaging channel (easiest to set up) |

---

## What Was Ruled Out

| Option | Reason |
|--------|--------|
| Local LLMs | Competes with agent RAM, cloud APIs cheap enough |
| M1 Mac Mini | Similar used price to M4 new, slower, less future-proof |
| M4 Pro 48GB | Only needed for 32B+ local models — overkill |
| M4 24GB | $400 more, only useful for local models |
| Gemini 3.1 Pro | Preview only, unstable, more expensive than 2.5 Pro |
| External SSD (now) | 512GB internal is sufficient to start |

---

## Next Steps

1. OpenClaw installation (Node.js 22+, CLI onboarding)
2. Telegram channel setup
3. OpenRouter + OpenClaw config
4. Crawl4AI skill setup
5. Security hardening
