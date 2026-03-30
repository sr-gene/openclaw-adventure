# Mac Mini Spec Research for OpenClaw

## What is OpenClaw?

OpenClaw is a self-hosted personal AI agent gateway ([github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)) that runs as a persistent daemon connecting LLMs to your messaging apps and local system. Key capabilities:

- Responds via WhatsApp, Telegram, iMessage, Discord, Slack, and 20+ other platforms
- Executes shell commands, manages files, automates browser tasks
- Runs scheduled tasks (cron), handles webhooks, controls a Chrome/Chromium instance
- Extends via "skills" from ClawHub (public skill registry)

**Software requirements:** Node.js 22+, macOS/Linux/WSL2.

---

## Use Case

Financial research and strategy using **multiple concurrent agents** doing web crawling, data gathering, and analysis — all routed to cloud LLMs. No local models.

---

## Hardware Decision: Mac Mini M4, 16GB RAM, 512GB SSD

**PURCHASED**

| Component | Choice |
|-----------|--------|
| Mac Mini M4 16GB 512GB | New |

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

### What was ruled out
| Option | Reason |
|--------|--------|
| M4 24GB | $400 more — only useful for local models, which we ruled out |
| M4 Pro 48GB | Only needed for 32B–70B local models |
| M1 used 16GB | Too close in price, slower, shorter support window |

---

## LLM Stack (Cloud APIs Only)

Accessed via **OpenRouter** (single API key, native OpenClaw integration):

| Role | Model | Cost/1M tokens |
|------|-------|----------------|
| Strategy & synthesis | Claude Sonnet 4.6 | $3 in / $15 out |
| Web research + search | GPT-4o | $2.50 in / $10 out |
| Mid-tier analysis | Gemini 2.5 Pro | $1.25 in / $5 out |
| Bulk crawl & summarize | DeepSeek V3.2 | $0.28 in / $0.42 out |

**Note:** Use Gemini 2.5 Pro, not 3.1 Pro (3.1 is preview-only, more expensive, unstable).

### Monthly Cost Estimate

| Usage | Cost |
|-------|------|
| Light (5–10 tasks/day) | ~$55/mo |
| Medium (20–30 tasks/day) | ~$161/mo |
| Heavy (50+ tasks/day) | ~$320/mo |

Cost tip: Route 90% of tasks to DeepSeek, use Claude/GPT-4o for final synthesis only.

---

## Web Crawling

- **Crawl4AI** — primary, local-first, LLM-ready markdown output
- **Firecrawl** — managed API fallback

---

## Next Steps

1. OpenClaw installation (Node.js 22+, CLI onboarding)
2. Telegram channel setup
3. OpenRouter + OpenClaw config
4. Crawl4AI skill setup
5. Security hardening
