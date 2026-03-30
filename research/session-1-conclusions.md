# Session 1: Research Conclusions

## Hardware Purchased

**Mac Mini M4, 16GB RAM, 512GB SSD**

- 512GB internal SSD — enough for macOS + OpenClaw + apps + research data
- No external SSD needed immediately (512GB gives plenty of room)
- Can still add external 1TB SSD later if storage fills up
- No local LLMs — 16GB RAM is sufficient for multi-agent cloud API usage

---

## Use Case

Financial research and strategy automation using OpenClaw multi-agent setup:
- Multiple agents run in parallel doing web crawling and data gathering
- All LLM inference via cloud APIs (no local models)
- Results delivered via messaging app (Telegram)

---

## LLM Stack

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

| Option | Reason ruled out |
|--------|-----------------|
| Local LLMs | Competes with agent RAM, cloud APIs cheap enough |
| M1 Mac Mini | Similar used price to M4 new, slower, less future-proof |
| M4 Pro 48GB | Only needed for 32B+ local models — overkill |
| 24GB RAM | $400 more, only useful for local models |
| Gemini 3.1 Pro | Preview only, unstable, more expensive than 2.5 Pro |
| External SSD (now) | 512GB internal is sufficient to start |

---

## Next: OpenClaw Setup
