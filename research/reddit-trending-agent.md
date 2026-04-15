# UC-8: Reddit Trending Intelligence Agent

Research date: 2026-04-10
OpenClaw version target: 2026.3.x

---

## 1. What This Is

An OpenClaw agent that monitors AI-related subreddits for trending posts and delivers intelligence to Telegram. It combines two proven OpenClaw patterns:

- **Heartbeat pattern** (like UC-6) — polls subreddits every 30 minutes, silently ignores noise, and only alerts when a post crosses the trending threshold
- **Cron pattern** (like UC-5) — sends a daily 7 AM digest of the top posts from each subreddit

The agent uses the Reddit Official API for data access, relative engagement thresholds for trending detection (auto-adapts to community size), and DeepSeek V3.2 for AI summarization.

---

## 2. Subreddits Monitored

### Starting set

| Subreddit | Topic | Approx. Size | Notes |
|-----------|-------|-------------|-------|
| r/ClaudeAI | Claude, Anthropic ecosystem | ~180K members | Primary |
| r/ClaudeCode | Claude Code CLI tool | Dedicated community | Primary |
| r/OpenClaw | OpenClaw community, tips, use cases | Smaller | Primary |

### Future expansion (after initial tuning)

| Subreddit | Topic | When to add |
|-----------|-------|-------------|
| r/LocalLLaMA | Local model community (Ollama, llama.cpp) | After initial 3 subreddits are stable |
| r/MachineLearning | Academic/industry ML research | If AI research tracking is desired |
| r/artificial | General AI discussion | If broader AI coverage is needed |
| r/ChatGPT / r/OpenAI | OpenAI ecosystem | For competitor intelligence |

---

## 3. OpenClaw Skill Setup

### Step 1: Search ClawHub for existing Reddit skills

```bash
# Check if community Reddit skills exist
openclaw skills search "reddit"
clawhub search "reddit"

# Also check broader social media skills
openclaw skills search "social media"
clawhub search "social monitor"
```

**If a suitable skill exists** (e.g., `reddit-monitor`, `reddit-scraper`):

```bash
openclaw skills install reddit-monitor
# or via clawhub:
clawhub install reddit-monitor
```

After installing, start a new OpenClaw session to activate it.

**If no suitable skill exists** — create a custom skill:

```bash
# Create custom skill directory
mkdir -p ~/openclaw/skills/reddit-monitor
```

The custom skill would consist of:
- `skill.md` — prompt-based skill definition that instructs the agent how to use PRAW
- A Python helper script wrapping PRAW for Reddit API calls
- Configuration for OAuth2 credentials

**Security note:** Stick to official `openclaw/*` namespace or well-reviewed community skills. Per the ecosystem tools research (~3% of ClawHub's 13,000+ skills were flagged as malicious in early 2026).

### Step 2: Install PRAW (if custom skill)

```bash
# Install PRAW globally or in OpenClaw's Python environment
pip install praw
```

---

## 4. Reddit API Authentication

### Register OAuth app

1. Go to [reddit.com/prefs/apps](https://reddit.com/prefs/apps)
2. Click "create another app..."
3. Select **"script"** type (for personal use)
4. Fill in:
   - Name: `openclaw-reddit-monitor`
   - Redirect URI: `http://localhost:8080` (not used for script type)
5. Submit and note the **client_id** (under the app name) and **client_secret**

**Important:** Reddit requires approval for new API applications as of 2024. Self-serve registration was removed. Submit and wait — may take days to weeks.

### Store credentials in OpenClaw

```bash
# Add to ~/.openclaw/.env (keep out of skill files)
REDDIT_CLIENT_ID=your_client_id
REDDIT_CLIENT_SECRET=your_client_secret
REDDIT_USERNAME=your_reddit_username
REDDIT_PASSWORD=your_reddit_password
REDDIT_USER_AGENT=openclaw-reddit-monitor/1.0
```

### Rate limits

| Tier | Limit | Notes |
|------|-------|-------|
| Authenticated | 60 req/min | Sufficient for 3 subreddits at 30 min intervals |
| Pagination cap | 1,000 posts per query | More than enough for top/hot listings |
| Free tier | Non-commercial personal use | No cost |

---

## 5. Agent Configuration — Real-time Alerts (Heartbeat Pattern)

Following UC-6's `HEARTBEAT.md` approach. The agent checks subreddits on a schedule, stays silent when nothing is trending, and only alerts when a post crosses the threshold.

### Create heartbeat checklist

File: `~/openclaw/workspace/REDDIT_MONITOR.md`

```md
# Reddit Trending Monitor

## Subreddit Checks
- [ ] Check r/ClaudeAI for new posts in the last 30 min. Fetch the subreddit's last 24h posts to establish a baseline (median upvotes and comments). If any new post's engagement is in the top 10% relative to that baseline, summarize it (2-3 sentences) and alert via Telegram with title, link, upvotes, comment count, and summary.
- [ ] Check r/ClaudeCode for new posts in the last 30 min. Same threshold logic as above.
- [ ] Check r/OpenClaw for new posts in the last 30 min. Same threshold logic as above.
```

### Configure heartbeat in openclaw.json

Add to `~/.openclaw/openclaw.json`:

```json5
{
  agents: {
    defaults: {
      redditMonitor: {
        every: "30m",                    // check every 30 minutes
        target: "telegram",              // send alerts to Telegram
        lightContext: true,              // only load REDDIT_MONITOR.md, not full context
        isolatedSession: true,           // fresh session each run (saves tokens)
        includeReasoning: false,         // clean alerts, no chain-of-thought
        activeHours: {
          start: "00:00",               // monitor 24/7
          end: "23:59",
          timezone: "Asia/Seoul"
        }
      }
    }
  }
}
```

### Alert behavior

- Nothing trending → agent replies `HEARTBEAT_OK` → **suppressed, you see nothing**
- Post crosses threshold → agent sends formatted alert to Telegram → **you get notified**
- Replies ≤ 300 chars with `HEARTBEAT_OK` are treated as acknowledgments and hidden

### Cost-saving settings

| Setting | What it does | Why |
|---------|-------------|-----|
| `lightContext: true` | Only loads REDDIT_MONITOR.md into context | Reduces from ~100K tokens to ~2-5K per run |
| `isolatedSession: true` | Fresh session, no conversation history | Avoids accumulating token cost |
| `includeReasoning: false` | Suppresses chain-of-thought | Cleaner alerts, fewer output tokens |

### Active hours options

```json5
// Option A: 24/7 monitoring (recommended — Reddit is global)
activeHours: { start: "00:00", end: "23:59", timezone: "Asia/Seoul" }

// Option B: Waking hours only (save cost)
activeHours: { start: "07:00", end: "23:00", timezone: "Asia/Seoul" }
```

---

## 6. Agent Configuration — Daily Digest (Cron Pattern)

Following UC-5's daily briefing at 7 AM pattern.

### Configure cron in openclaw.json

```json5
// Add to the cron array in ~/.openclaw/openclaw.json
{
  cron: [
    // ... existing UC-5 entries ...
    {
      schedule: "0 7 * * *",           // 7:00 AM daily
      timezone: "Asia/Seoul",
      task: "Generate Reddit daily digest: For each of r/ClaudeAI, r/ClaudeCode, and r/OpenClaw, fetch the top posts from the last 24 hours. Rank by relative engagement (upvotes + comments vs subreddit baseline). For the top 5 posts per subreddit, include: title, Reddit link, upvote count, comment count, and a one-line summary. Format as a clean digest and send to Telegram.",
      channel: "telegram"
    }
  ]
}
```

---

## 7. Trending Detection Algorithm

The agent evaluates trending status **relative to each subreddit's own baseline**, not using absolute numbers. This auto-adapts to community size.

### How it works

1. **Establish baseline:** Fetch the subreddit's posts from the last 24 hours (via `subreddit.top(time_filter='day')` or `subreddit.hot(limit=100)`)
2. **Calculate median engagement:** Compute the median upvote count and median comment count for the baseline set
3. **Score new posts:** For each new post found in the heartbeat check, compare its engagement to the baseline
4. **Trigger threshold:** Alert if a post is in the **top 10%** of the baseline distribution (configurable)

### Why relative thresholds

| Subreddit | Median upvotes (24h) | Top 10% threshold | Example "trending" post |
|-----------|---------------------|-------------------|------------------------|
| r/ClaudeAI (~180K) | ~50 | ~200+ | Major feature announcement |
| r/ClaudeCode | ~15 | ~60+ | Useful workflow tip goes viral |
| r/OpenClaw (smaller) | ~5 | ~20+ | New skill release gets attention |

Absolute thresholds would miss everything on r/OpenClaw while drowning in r/ClaudeAI noise.

### Implementation note

The trending logic lives **in the agent prompt** (REDDIT_MONITOR.md), not in a hard-coded script. The LLM evaluates relative engagement based on the baseline data it fetches. This makes tuning easy — just update the markdown instructions.

---

## 8. Output Formats

### Real-time alert (heartbeat trigger)

Sent immediately to Telegram when a post crosses the trending threshold:

```
🔥 Trending on r/ClaudeAI

**Claude Code now supports MCP server chaining**
🔗 https://reddit.com/r/ClaudeAI/comments/...
⬆️ 342 upvotes | 💬 89 comments

Summary: Users report that the latest Claude Code update enables
chaining multiple MCP servers in a single session. Early feedback
is positive, with developers sharing complex multi-tool workflows.
```

### Daily 7 AM digest

Sent once daily via cron:

```
📊 Reddit AI Daily — 2026-04-10

## r/ClaudeAI (top 5)
1. Claude Code now supports MCP server chaining — ⬆️ 342 | 💬 89
   First-party MCP chaining landed, enabling multi-tool workflows
2. Sonnet 4.6 benchmarks show 15% improvement — ⬆️ 218 | 💬 56
   Community benchmarks confirm speed gains on coding tasks
3. ...

## r/ClaudeCode (top 5)
1. My workflow: Claude Code + git worktrees — ⬆️ 67 | 💬 23
   Developer shares isolation strategy using worktree agents
2. ...

## r/OpenClaw (top 5)
1. New Firecrawl skill v2 released — ⬆️ 28 | 💬 12
   Major update adds JS rendering and anti-bot handling
2. ...
```

---

## 9. Model Selection

Heartbeat runs frequently (48 times/day), so cost matters. Following UC-6's analysis pattern:

| Model | Cost per heartbeat | Monthly (48 runs/day) | Best for |
|-------|-------------------|----------------------|----------|
| **DeepSeek V3.2** | ~$0.001 | ~$1.50/month | Heartbeat checks + summarization |
| **Haiku** | ~$0.001 | ~$1.50/month | Simple trending detection only |
| **Sonnet** | ~$0.005 | ~$7.50/month | Complex analysis (overkill for this) |

**Decision:** Use **DeepSeek V3.2** for both heartbeat monitoring and digest generation. Matches the "let DeepSeek handle 90% of work" rule from the LLM cost strategy.

### Monthly cost estimate

| Component | Est. cost |
|-----------|----------|
| Reddit API | Free (personal non-commercial) |
| DeepSeek — heartbeat (48 runs/day × 30 days) | ~$2-3/mo |
| DeepSeek — daily digest (1 run/day × 30 days) | ~$1-2/mo |
| DeepSeek — AI summaries (~75 posts/day) | ~$2-3/mo |
| **Total** | **~$5-8/mo additional** |

---

## 10. Non-Goals

- No sentiment analysis — simple AI summary only (keeps cost down)
- No comment thread deep-dives — post-level monitoring only
- No web UI or dashboard — Telegram-only delivery
- Not replacing UC-1 (Twitter/X Intelligence) — separate use case, separate infrastructure
- No real-time PRAW streaming in v1 — heartbeat polling is simpler and cheaper to start

---

## 11. Open Questions

| Item | Decision needed |
|------|----------------|
| ClawHub Reddit skill | Does a community Reddit skill exist? Search `clawhub search "reddit"` before building custom |
| Heartbeat vs. PRAW stream | Heartbeat (poll every 30 min) is simpler/cheaper; PRAW streaming is more responsive. Start with heartbeat, upgrade later if latency matters |
| Trending percentile | Start at top 10%? Top 5%? Needs tuning after launch based on alert volume |
| Reddit API approval | Submit application — unknown wait time (days to weeks) |
| Fallback if API rejected | Apify Reddit Scraper skill as backup (~$5/mo) — search `openclaw skills search "apify"` |
| Deduplication | How to avoid re-alerting on same post across heartbeat runs (track post IDs in workspace file?) |
| Subreddit expansion | When and how to add more subreddits (edit REDDIT_MONITOR.md + cron task) |
| Dedicated Telegram channel | Separate from UC-5 briefings, or shared channel with topic prefixes? |

---

## 12. Implementation Checklist

| Step | Action | Depends on |
|------|--------|-----------|
| 1 | Submit Reddit OAuth app for approval | — |
| 2 | Search ClawHub for existing Reddit skill | — |
| 3 | Install skill or create custom PRAW wrapper | Steps 1 + 2 |
| 4 | Create `REDDIT_MONITOR.md` heartbeat checklist | Step 3 |
| 5 | Add heartbeat config to `openclaw.json` | Step 4 |
| 6 | Add daily digest cron to `openclaw.json` | Step 3 |
| 7 | Test with one subreddit (r/ClaudeAI) | Steps 5 + 6 |
| 8 | Tune trending threshold (10% → adjust) | Step 7 |
| 9 | Enable all 3 subreddits | Step 8 |
| 10 | Monitor cost for 1 week, adjust if needed | Step 9 |
