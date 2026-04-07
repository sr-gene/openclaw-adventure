# Plan: Gemma 4 E4B Local Setup + OpenClaw Integration

> **Target machine:** Mac Mini M4, 24GB RAM — do NOT run on 16GB Mac.
> Pull this repo on the Mac Mini, then follow these steps.

---

## Context

- Gemma 4 E4B: ~5GB at Q4, runs at 40–60 tok/s on M-series, beats Gemma 3 27B on benchmarks
- Goal: replace DeepSeek V3.2 cloud subagent with free local inference via Ollama
- DeepSeek stays as automatic fallback when Ollama is unavailable
- 24GB RAM budget: macOS (~3GB) + OpenClaw + 4 agents (~6GB) + Ollama E4B (~5GB) ≈ 14GB — comfortable headroom

---

## Pre-flight: Verify OpenClaw Ollama Support

**Do this first before touching any config.**

```bash
# Check actual OpenClaw docs for Ollama provider support
open https://docs.openclaw.ai/providers/ollama

# Or check via CLI
openclaw --help | grep -i ollama
openclaw providers list
```

**What to confirm:**
1. Does OpenClaw support an `ollama` provider block in `openclaw.json`?
2. What is the exact model string format? (`ollama/gemma4:e4b` or `local/gemma4:e4b` or something else?)
3. Is `fallback` a supported key in the `models` config block?
4. Does cross-provider fallback (Ollama → OpenRouter) work automatically?

If Ollama provider is not supported natively, see the **Fallback Architecture** section at the bottom.

---

## Step 1 — Install Ollama + Pull Model

```bash
brew install ollama
ollama pull gemma4:e4b      # ~5GB download
ollama run gemma4:e4b "summarize: Apple reported record Q1 earnings"  # smoke test
```

---

## Step 2 — Run Ollama as Persistent LaunchAgent

```bash
brew services start ollama

# Verify it survives reboot
brew services list | grep ollama   # should show: started
```

**Optional: Set idle unload to free RAM when not in use**

```bash
# Ollama auto-unloads model after this timeout (default: 5m)
# Add to ~/.zshrc or LaunchAgent environment:
export OLLAMA_KEEP_ALIVE=5m
```

This means E4B only consumes ~5GB during active inference, not 24/7.

---

## Step 3 — Update OpenClaw Config

Edit `~/.openclaw/openclaw.json`:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    FIRECRAWL_API_KEY: "fc-..."
  },

  gateway: {
    port: 18789,
    bind: "loopback"
  },

  providers: {
    openrouter: {
      apiKey: "${OPENROUTER_API_KEY}"
    },
    ollama: {
      baseUrl: "http://localhost:11434"   // verify key name against docs
    }
  },

  models: {
    primary:   "openrouter/anthropic/claude-sonnet-4-6",
    subagent:  "ollama/gemma4:e4b",              // verify model string format
    heartbeat: "openrouter/google/gemini-2.5-pro",
    fallback:  "openrouter/deepseek/deepseek-v3.2"  // verify `fallback` key exists
  },

  channels: {
    telegram: {
      enabled: true,
      botToken: "${TELEGRAM_BOT_TOKEN}",
      dmPolicy: "allowlist",
      allowFrom: ["YOUR_NUMERIC_TELEGRAM_ID"]
    }
  }
}
```

> All syntax marked with `// verify` must be confirmed against actual OpenClaw docs before applying.

---

## Step 4 — Update AGENTS.md

In `~/.openclaw/workspace/AGENTS.md`, update data-fetcher agents:

```markdown
## data-fetcher
model: ollama/gemma4:e4b    // verify model string format
```

> **Note:** The data-fetcher role runs in parallel (3–4 concurrent instances during fan-out).
> Ollama processes requests sequentially by default.
> If parallelism is critical, keep data-fetchers on DeepSeek and use Ollama only for ad-hoc Telegram queries.

---

## Step 5 — Mac Mini Never Sleeps

```bash
# Save current settings first (for rollback)
pmset -g > ~/pmset-backup.txt

# Apply always-on settings
sudo pmset -a sleep 0
sudo pmset -a disksleep 0
sudo pmset -a womp 1       # wake on network access

# Verify
pmset -g live
# Should show: sleep 0, disksleep 0, womp 1

# Also set in System Settings:
# System Settings → Energy → Prevent automatic sleeping when display is off ✓
```

**Rollback if needed:**
```bash
# Restore defaults
sudo pmset -a sleep 1
sudo pmset -a disksleep 10
```

---

## Step 6 — Restart OpenClaw + Validate

```bash
# Use stop + start (NOT restart — known macOS bug)
openclaw gateway stop
openclaw gateway start
openclaw gateway status
openclaw dashboard        # http://localhost:18789
```

**Validation checklist:**
- [ ] `ollama list` shows `gemma4:e4b`
- [ ] `brew services list` shows `ollama` as `started`
- [ ] `pmset -g live` shows `sleep 0`
- [ ] OpenClaw dashboard shows subagent resolving to Ollama
- [ ] Send test via Telegram: `"brief me"` — pipeline completes without error
- [ ] Stop Ollama (`brew services stop ollama`), send another task — confirm DeepSeek fallback kicks in (check OpenClaw logs)
- [ ] Run parallel pipeline (`"research AAPL Q1 2026"`), monitor RAM: `memory_pressure` should stay green

---

## Fallback Architecture (if Ollama provider not supported natively)

If OpenClaw doesn't support Ollama as a provider, use this split approach:

1. Keep pipeline subagents on DeepSeek (cloud, parallel-safe):
   ```json5
   models: {
     subagent: "openrouter/deepseek/deepseek-v3.2"
   }
   ```

2. Use Ollama separately for ad-hoc Telegram queries via a local HTTP proxy or
   OpenClaw's `local` provider (if it exists).

3. Or expose Ollama via an OpenAI-compatible endpoint and route through OpenRouter
   (not possible — OpenRouter only routes to their own providers).

---

## Cost Reference

| Stage | Model | Cost/run |
|---|---|---|
| Orchestration (×2) | Claude Sonnet 4.6 | ~$0.18 |
| **Data fetching (×2)** | **Ollama E4B (local)** | **$0.00** ← was $0.04 |
| Metrics analysis | Gemini 2.5 Pro | ~$0.13 |
| Risk assessment | Gemini 2.5 Pro | ~$0.09 |
| Thesis writing | Claude Sonnet 4.6 | ~$0.28 |
| **Total** | | **~$0.68** (was $0.72) |

Savings per run: $0.04. At 30 runs/day: ~$36/month saved.
Primary benefit is **privacy** (financial data never leaves the machine for subagent tasks).
