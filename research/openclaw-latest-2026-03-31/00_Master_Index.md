# OpenClaw Research — Master Index

**Research date:** 2026-03-31
**Context:** Mac Mini M4, 16GB RAM — financial research via multi-agent workflows, cloud LLMs via OpenRouter, Telegram channel

> **Source transparency:** WebFetch and WebSearch were unavailable in this research session. All content is sourced from workspace notes created 2026-03-30 (which cite official docs and community sources) plus model knowledge current to August 2025. Items not independently verified in live docs are marked [unverified].

---

## File Index

| File | Topic | Reading Time |
|------|-------|-------------|
| [01_Installation.md](./01_Installation.md) | Install methods, Node.js requirements, onboarding wizard, verification | ~5 min |
| [02_Configuration.md](./02_Configuration.md) | openclaw.json structure, OpenRouter, workspace files, macOS daemon + KeepAlive bug | ~8 min |
| [03_Multi_Agent_Setup.md](./03_Multi_Agent_Setup.md) | Pipeline architecture, AGENTS.md config, financial research pipeline example | ~10 min |
| [04_Telegram_Channel.md](./04_Telegram_Channel.md) | BotFather setup, bot token config, numeric ID, dmPolicy allowlist | ~5 min |
| [05_Security.md](./05_Security.md) | File permissions, API key storage, gateway binding, ClawHub vetting, audit command | ~6 min |
| [06_Thinking_Framework.md](./06_Thinking_Framework.md) | Decision trees, model selection, command reference, cost estimator | ~5 min |

---

## Recommended Reading Order

### "I want to set up OpenClaw right now"
1. [06_Thinking_Framework.md](./06_Thinking_Framework.md) — Setup Order checklist (last section)
2. [01_Installation.md](./01_Installation.md) — Install + onboard
3. [04_Telegram_Channel.md](./04_Telegram_Channel.md) — Connect Telegram
4. [02_Configuration.md](./02_Configuration.md) — Configure OpenRouter + workspace files
5. [05_Security.md](./05_Security.md) — Harden immediately after setup
6. [03_Multi_Agent_Setup.md](./03_Multi_Agent_Setup.md) — Build your research pipeline

### "I want to understand the architecture first"
1. This file (Key Findings summary below)
2. [03_Multi_Agent_Setup.md](./03_Multi_Agent_Setup.md) — Pipeline design
3. [02_Configuration.md](./02_Configuration.md) — Full config reference
4. [06_Thinking_Framework.md](./06_Thinking_Framework.md) — Decision trees + model guide
5. [01_Installation.md](./01_Installation.md) — Then install
6. [04_Telegram_Channel.md](./04_Telegram_Channel.md) + [05_Security.md](./05_Security.md)

### "I'm troubleshooting a specific problem"
- Daemon issues → [02_Configuration.md §Daemon Management](./02_Configuration.md#daemon-management-on-macos)
- Telegram not working → [04_Telegram_Channel.md §Troubleshooting](./04_Telegram_Channel.md#troubleshooting)
- Security concerns → [05_Security.md](./05_Security.md)
- Model cost questions → [06_Thinking_Framework.md §Cost Estimator](./06_Thinking_Framework.md#monthly-cost-estimator)
- Which model to use → [06_Thinking_Framework.md §Decision Tree 2](./06_Thinking_Framework.md#decision-tree-2--which-model-to-use)

---

## Key Findings (5 Points)

1. **One-liner installer is the fastest path on Mac.** `curl -fsSL https://openclaw.ai/install.sh | bash` handles Homebrew, Node.js 24, and the CLI in one step. Node.js 22.14+ is the minimum; 24.x is recommended for Apple Silicon.

2. **`openclaw gateway restart` is broken on macOS.** A known launchd KeepAlive race condition leaves the daemon in a broken state. Always use `openclaw gateway stop && openclaw gateway start`. Apply the conditional KeepAlive plist fix immediately after setup (documented in `02_Configuration.md`).

3. **Route 90% of tasks to DeepSeek V3.2 via OpenRouter.** At $0.28/$0.42 per 1M tokens, DeepSeek is the workhorse for all scraping and summarizing. Claude Sonnet 4.6 ($3/$15) should only handle final synthesis. This keeps medium-usage costs around $161/month vs ~$800+ if Claude handles everything.

4. **Always use `dmPolicy: "allowlist"` with your numeric Telegram ID.** `"pairing"` mode resets on every daemon restart — unreliable for 24/7 operation. `"open"` exposes your agent to anyone who finds your bot username. Get your numeric ID from `openclaw logs --follow` after sending your bot a first message.

5. **Firecrawl is the documented web research skill; Crawl4AI has no confirmed official ClawHub skill.** Firecrawl routes through a remote browser (no local Chromium RAM overhead), handles 403s, and supports parallel sessions — ideal for the Mac Mini M4 memory-conscious setup. Install via `clawhub install firecrawl`.

---

## Glossary

| Term | Definition |
|------|------------|
| **OpenClaw** | Self-hosted personal AI agent gateway. Runs as a daemon, connects LLMs to messaging platforms and local system. GitHub: [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw) |
| **ClawHub** | OpenClaw's public skill registry. Skills extend OpenClaw with new capabilities (web scraping, file ops, browser control). Analogous to npm for OpenClaw capabilities. |
| **Daemon** | The OpenClaw background process. Installed as a macOS LaunchAgent at `~/Library/LaunchAgents/com.openclaw.gateway.plist`. |
| **LaunchAgent** | macOS mechanism for user-level background processes. Starts at login, runs under your user account. Managed by `launchctl`. |
| **KeepAlive** | A LaunchAgent plist setting that tells launchd to auto-restart a process if it exits. OpenClaw's default `<true/>` setting causes a bug on restart — must be changed to conditional. |
| **OpenRouter** | Unified LLM API proxy. One API key routes to Claude, GPT-4o, Gemini, DeepSeek, and 200+ other models. [openrouter.ai](https://openrouter.ai) |
| **SOUL.md** | Workspace file injected as system prompt every session. Defines the agent's persistent identity and behavior. |
| **USER.md** | Workspace file describing the user — timezone, goals, preferences. Agent reads this to personalize responses. |
| **AGENTS.md** | Workspace file defining multi-agent pipelines, sub-agent roles, and orchestration logic. |
| **HEARTBEAT.md** | Workspace file defining scheduled autonomous tasks (cron-like). Drives proactive agent behaviors. |
| **dmPolicy** | Telegram channel setting controlling who can message the bot. Values: `allowlist`, `pairing`, `open`. Always use `allowlist`. |
| **allowFrom** | Array of numeric Telegram user IDs permitted to message the bot when `dmPolicy` is `"allowlist"`. |
| **Orchestrator** | The top-level agent in a multi-agent pipeline. Decomposes tasks, assigns sub-agents, merges results. |
| **Sub-agent** | A scoped agent in a pipeline assigned a specific task and model. |
| **Firecrawl** | Web scraping service that uses a remote managed browser. ClawHub skill: `clawhub install firecrawl`. [firecrawl.dev](https://firecrawl.dev) |
| **Gateway** | OpenClaw's local HTTP server (default port 18789). Handles all LLM and channel traffic. Should be bound to loopback only. |

---

## Source Documents

### Primary workspace sources (verified in this session)
- `research/getting-started-openclaw-2026-03-30.md`
- `research/setup-guide-2026-03-30.md`
- `research/mac-mini-specs-2026-03-30.md`

### Referenced external sources (from workspace notes — links not verified in this session)
- [docs.openclaw.ai](https://docs.openclaw.ai) — Official documentation
- [docs.openclaw.ai/install](https://docs.openclaw.ai/install) — Installation guide
- [docs.openclaw.ai/providers/openrouter](https://docs.openclaw.ai/providers/openrouter) — OpenRouter integration
- [docs.openclaw.ai/channels/telegram](https://docs.openclaw.ai/channels/telegram) — Telegram channel setup
- [docs.openclaw.ai/gateway/security](https://docs.openclaw.ai/gateway/security) — Security hardening
- [docs.openclaw.ai/multi-agent](https://docs.openclaw.ai/multi-agent) — Multi-agent documentation
- [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw) — Source repository
- [openrouter.ai/docs/guides/guides/openclaw-integration](https://openrouter.ai/docs/guides/guides/openclaw-integration) — OpenRouter + OpenClaw
- [docs.firecrawl.dev/developer-guides/openclaw](https://docs.firecrawl.dev/developer-guides/openclaw) — Firecrawl + OpenClaw
- [github.com/TechNickAI/openclaw-config](https://github.com/TechNickAI/openclaw-config) — Community config example
- [github.com/digitalknk/openclaw-runbook](https://github.com/digitalknk/openclaw-runbook) — Community operational runbook
