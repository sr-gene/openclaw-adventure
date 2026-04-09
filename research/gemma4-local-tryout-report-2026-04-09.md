# Gemma4 Local Inference Integration Report

**Date:** 2026-04-07 ~ 2026-04-09  
**System:** Mac Mini M4, 16GB RAM  
**OpenClaw Version:** 2026.4.5  
**Ollama Version:** 0.20.2  
**Status:** Trial completed; reverted to cloud-based inference

## Objective

Replace DeepSeek cloud subagent with free local inference via Ollama to reduce API costs and latency in OpenClaw multi-agent workflows.

## Executive Summary

The trial to integrate Gemma4 (via Ollama) as a local inference backend for OpenClaw proceeded through six phases: installation, configuration, timeout resolution, performance testing, capability testing, and cron job validation. While installation and basic configuration succeeded, the trial revealed fundamental limitations in Gemma4's agentic behavior. Both Gemma4:e2b and Gemma4:e4b lack the autonomous tool-execution capability required for OpenClaw workflows—they understand tool definitions but refuse to execute tools without explicit user confirmation. The 16GB RAM system also experienced significant memory pressure with the larger models, limiting practical deployment. **Final decision: revert to `openrouter/deepseek/deepseek-chat` for production use.**

---

## Phase 1: Installation

### Ollama Setup

```bash
brew install ollama
brew services start ollama
```

**Result:** Ollama v0.20.2 installed and running as persistent LaunchAgent.

### Model Pulling

```bash
ollama pull gemma4:e4b
ollama pull gemma4:e2b
```

- **gemma4:e4b** (9-parameter variant, Q4_K_M quantization): 9.6 GB downloaded, ~9.6 GB RAM loaded
- **gemma4:e2b** (2-parameter variant, MoE architecture): 7.2 GB downloaded, ~7.7 GB RAM loaded

### Smoke Test

```bash
curl http://localhost:11434/api/generate -d '{"model": "gemma4:e4b", "prompt": "Hello there!", "stream": false}'
```

**Result:** Response generated at 11.89 tok/s. Basic model loading and inference confirmed working.

---

## Phase 2: OpenClaw Configuration

Three distinct approaches were attempted, each revealing different failure modes.

### Attempt 1: Ollama Plugin + Provider Collision

**Configuration:**
```yaml
models:
  providers:
    ollama:
      baseUrl: "http://localhost:11434"
      api: "ollama"

plugins:
  ollama:
    enabled: true
```

**Issue:** Name collision between the `ollama` plugin (which registers its own `text-inference: ollama` capability) and the `models.providers.ollama` entry.

**Error:**
```
Unknown model: ollama/gemma4:e4b
```

**Root Cause:** The plugin's internal capability registration conflicted with the provider definition, causing model discovery to fail.

**Lesson:** Plugin and provider names must not overlap. The plugin already provides `text-inference` functionality; adding a `models.providers.ollama` entry creates a conflict.

### Attempt 2: OpenAI-Compatible Endpoint (`/v1`)

**Configuration:**
```yaml
models:
  providers:
    local:
      baseUrl: "http://localhost:11434/v1"
      api: "openai-completions"

plugins:
  ollama:
    enabled: false  # Disabled to avoid conflicts
```

**Result:** Model responded to requests. Config validation passed.

**Critical Issue:** Tool calling was broken.

**Explanation:** Ollama's `/v1` OpenAI-compatible endpoint strips Gemma4's native tool-calling tokens. OpenClaw attempted to send tool schemas to the model, but the endpoint transformed the request into a format that removed tool-calling signal. Official OpenClaw documentation explicitly states: **"Do NOT use the /v1 OpenAI-compatible URL. This breaks tool calling."**

**Lesson:** Local model APIs must preserve the native tool schema format. The OpenAI-compat wrapper is insufficient for models with native tool-calling support.

### Attempt 3: Native Ollama API (Correct Configuration)

**Configuration:**
```yaml
models:
  providers:
    local:
      baseUrl: "http://localhost:11434"
      api: "ollama"

plugins:
  ollama:
    enabled: true
```

**Result:** Configuration validated successfully, no startup errors, model connectivity confirmed.

**Outcome:** While the configuration was technically correct per OpenClaw documentation, Gemma4's model behavior remained problematic (see Phase 5).

**Lesson:** Correct configuration is necessary but not sufficient—model capability constraints cannot be bypassed through endpoint selection alone.

---

## Phase 3: Timeout Tuning

Early inference attempts timed out with default OpenClaw settings. Investigation revealed two distinct timeout parameters with different implications.

### Initial Problem

- Default `agents.defaults.timeoutSeconds: ~60` caused model responses to abort mid-generation
- Adjusting this alone did not fully resolve the issue

### Root Cause Discovery

The real bottleneck was `llm.idleTimeoutSeconds`—the maximum time to wait for the first token from the model. On a 16GB system with 9.6GB consumed by gemma4:e4b, the first token can take 20–50 seconds, especially if the model was previously unloaded.

### Configuration Solution

```yaml
llm:
  idleTimeoutSeconds: 300  # 5 minutes for first token
agents:
  defaults:
    timeoutSeconds: 600    # 10 minutes total inference
```

**Environment Variable:**
```bash
export OLLAMA_KEEP_ALIVE=-1
```

Setting `OLLAMA_KEEP_ALIVE=-1` prevents Ollama from automatically unloading the model after 5 minutes of inactivity, reducing cold-start latency on subsequent inference runs.

### Result

Timeout errors resolved. Model loaded predictably once first-token timeout was increased.

**Lesson:** Local model inference requires longer timeouts, especially on memory-constrained systems. The idle-timeout parameter controls time-to-first-token, not total generation time.

---

## Phase 4: Performance Testing

Benchmarked both Gemma4 variants on typical inference tasks.

### Methodology

- **Cold Start:** Model not in memory; Ollama must load from disk
- **Warm Start:** Model already loaded in memory
- **Task:** Simple instruction-following prompt (~50 tokens response)

### Results

| Model | Size on Disk | RAM Loaded | Cold Start | Warm Start | Tokens/sec |
|-------|--------------|------------|------------|------------|-----------|
| gemma4:e4b | 9.6 GB | ~9.6 GB | 47s | ~12 tok/s | 8–12 |
| gemma4:e2b | 7.2 GB | ~7.7 GB | 17s | ~8s | 8–10 |

### Memory Pressure Analysis

**With gemma4:e4b loaded:**
- System free memory: 14% (~640 MB free on 16GB system)
- Observed swap activity: 2.4M swapped to disk
- Performance degradation due to memory thrashing

**With gemma4:e2b loaded:**
- System free memory: ~25%
- Swap activity: minimal
- More stable but still memory-constrained

### Key Finding

16GB RAM is insufficient for production Gemma4 inference. The original plan's recommendation of 24GB is validated. With e4b loaded, the system enters heavy memory pressure, causing observable latency spikes and potential instability under sustained load.

---

## Phase 5: Agent Capability Testing

The configuration issues resolved in Phase 2–3, but a more fundamental problem emerged: Gemma4 models lack the autonomous tool-execution behavior required for OpenClaw's agentic workflows.

### Test 1: gemma4:e2b — Basic Tool Calling

**Setup:** Native Ollama API, tools defined in OpenClaw context

**Prompt:** "Check my email and summarize today's messages"

**Response:**
```
I am an AI language model and I do not have the capability to execute external commands. 
However, I can help you understand what commands or tools might be needed...
```

**Result:** Model explicitly refused to execute tools, despite having tool definitions in context.

### Test 2: gemma4:e4b — OpenAI-Compatible Endpoint

**Setup:** `/v1` endpoint with `openai-completions` API

**Prompt:** "Execute the calendar sync tool"

**Response:**
```
I'm unable to execute external tools or commands. I'm a language model designed to process 
and generate text based on the information provided to me.
```

**Result:** Same refusal pattern as e2b. The larger model size did not improve tool-execution behavior.

### Test 3: gemma4:e4b — Native Ollama API with Tool Tokens Preserved

**Setup:** Native Ollama API (`api: "ollama"`), tool-calling signals intact

**Prompt:** "Sync my calendar and email"

**Response:**
```
Before I proceed, I must confirm that you authorize me to access your calendar and email. 
These are sensitive resources. I recommend that you review the requested permissions and 
confirm explicitly before proceeding.

Let me outline what would happen:
1. Calendar sync would access your iCloud/Google Calendar
2. Email sync would access your Gmail account
3. ...
```

**Result:** Model acknowledged tool availability but asked for explicit permission instead of autonomously executing. Produced a planning/advisory response rather than tool invocation.

### Root Cause Analysis

Gemma4 models are instruction-tuned for helpful, cautious behavior. This design choice—intended to make them safe and user-friendly—makes them unsuitable for autonomous agentic workflows where the agent must execute tools independently without asking permission at each step. The model's personality is fundamentally at odds with the OpenClaw use case.

---

## Phase 6: Cron Job Testing

### Daily Morning Briefing (9 AM KST)

**Workflow:** Fetch Gmail messages → Execute `gog/icalendar-sync` tool → Summarize calendar events

**Configuration:** gemma4:e4b with native Ollama API (correct tool-calling setup)

**Result:** Task initiated but failed at tool execution.

**Model Response:**
```
I cannot execute external commands. The icalendar-sync tool is not available for me to use.
```

**Outcome:** Cron job ran without errors but produced no useful output. User would see empty briefing.

### Weekly Review (10 PM Sunday KST)

**Workflow:** Summarize completed tasks → Generate insight from logs

**Configuration:** gemma4:e2b

**Result:** Summary generated but tool-dependent features (reading task metadata from OpenClaw runtime) failed.

**Finding:** Even non-tool tasks showed degraded performance due to memory constraints and model capability mismatch.

---

## Configuration Details

### Final (Correct) Ollama Configuration

This configuration was verified to be syntactically correct per OpenClaw docs, though it did not resolve Gemma4's agentic limitations:

```yaml
models:
  providers:
    local:
      baseUrl: "http://localhost:11434"
      api: "ollama"

llm:
  idleTimeoutSeconds: 300
  
agents:
  defaults:
    timeoutSeconds: 600

plugins:
  ollama:
    enabled: true
```

### Environment Setup

```bash
# ~/.zshrc or ~/.bash_profile
export OLLAMA_KEEP_ALIVE=-1
```

### Launch Agent Configuration

```xml
<!-- ~/Library/LaunchAgents/com.ollama.user.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.ollama.user</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/local/bin/ollama</string>
    <string>serve</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>StandardOutPath</key>
  <string>/tmp/ollama.log</string>
  <key>StandardErrorPath</key>
  <string>/tmp/ollama.log</string>
</dict>
</plist>
```

---

## Key Learnings

### 1. Configuration: Use Native Ollama API

**Rule:** Use `http://localhost:11434` with `api: "ollama"`, NOT `http://localhost:11434/v1` with `api: "openai-completions"`.

The `/v1` OpenAI-compatible endpoint strips Gemma4's native tool-calling tokens, breaking tool execution entirely. This is documented in OpenClaw's official guidance and confirmed through testing.

### 2. Naming: Avoid Plugin-Provider Conflicts

**Rule:** Do not name a `models.providers` entry "ollama" if the Ollama plugin is enabled.

The plugin registers its own `text-inference: ollama` capability. Naming a provider the same causes a name collision. Use a unique name like `local` or `ollama-local`.

### 3. Timeouts: First-Token Latency Dominates

**Rule:** Set `llm.idleTimeoutSeconds` significantly higher than `agents.defaults.timeoutSeconds` on memory-constrained systems.

On 16GB RAM with 9.6GB model loaded, time-to-first-token routinely exceeds 30 seconds. The `idleTimeoutSeconds` parameter controls this critical path, not total inference duration.

### 4. Hardware: 16GB RAM is Insufficient

**Finding:** Plan specifications calling for 24GB RAM are validated by this trial.

With gemma4:e4b (9.6GB), system memory drops to 14% free, triggering heavy swap usage and latency spikes. gemma4:e2b (7.7GB) is more stable but still memory-constrained. 24GB provides 8GB+ free headroom for OS, Ollama overhead, and other services.

### 5. Model Limitations: Gemma4 Lacks Agentic Behavior

**Finding:** Both Gemma4:e2b and Gemma4:e4b are instruction-tuned to be helpful and cautious, which is inappropriate for autonomous agent workflows.

- **e2b:** Explicitly refuses tool execution ("I do not have the capability...")
- **e4b:** Asks for permission before executing tools rather than acting autonomously
- Neither model has the aggressive, tool-first behavior required for OpenClaw workflows

DeepSeek V3, by contrast, understands that it operates within a tool-rich environment and executes tools as the primary action, with advisory text only when necessary.

### 6. Cost-Benefit: Local Inference Breaks Even at ~$0.04/run

**Analysis:**
- Gemma4 inference cost: $0 (local) but requires 16GB+ system, 24h uptime, electricity
- DeepSeek Cloud cost: ~$0.04 per typical OpenClaw run (API token usage ~3000–5000 tokens)
- Break-even: ~6,000 runs before cloud inference cost exceeds local system cost (electricity, hardware depreciation)
- Conclusion: At current usage (~1 briefing + 1 review = 2 runs/week), cloud-based inference is more cost-effective

### 7. Context7 Documentation Reference

Library ID `/websites/openclaw_ai` contains 5,048 code snippets covering OpenClaw configuration, plugin setup, and model provider integration. This was the primary reference for resolving configuration issues.

---

## Decision and Recommendations

### Final Decision

**Revert to `openrouter/deepseek/deepseek-chat` for all production tasks.**

Rationale:
1. Gemma4 models lack autonomous tool-execution behavior
2. 16GB RAM insufficient for gemma4:e4b; 24GB systems would still be marginal with larger models
3. DeepSeek cost (~$0.04/run) is economical at current usage levels
4. Reliability and capability requirements outweigh local inference cost savings

### Recommendations for 24GB Mac Mini (Future)

If hardware is upgraded to 24GB RAM:

1. **Retry gemma4:26b** (MoE architecture, 26B parameters)
   - Larger parameter count may improve agentic reasoning
   - Test with same configuration as e4b trial
   - Monitor memory usage; may push 24GB system near limits

2. **Keep DeepSeek as fallback**
   - Configure secondary provider for tool-heavy tasks
   - Use local model for simple summarization/analysis tasks where tool calling is not required

3. **Preload model on boot**
   - Add model preload script to LaunchAgent to minimize cold-start latency
   - Set `OLLAMA_KEEP_ALIVE=-1` permanently in shell profile

4. **Monitor for agentic variants**
   - Google may release Gemma4 variants fine-tuned for autonomous tool use
   - Reassess capability periodically as models evolve

### Alternative Approach: Smaller Local Models for Non-Agentic Tasks

If local inference is desirable for cost, consider using smaller models (phi4, mistral, llama2) for tasks that do NOT require tool calling:
- Summarization (email, calendar, logs)
- Writing assistance (draft generation, editing)
- Classification (categorizing tasks, sentiment analysis)

Reserve DeepSeek for tool-execution workflows (automation, command execution, API calls).

---

## Current State (as of 2026-04-09)

### Active Configuration

- **Primary LLM:** `openrouter/deepseek/deepseek-chat`
- **Local Inference:** Ollama running with gemma4:e4b and gemma4:e2b available for manual testing
- **Status:** Not used in production workflows

### Cron Jobs

| Task | Schedule | Model | Status |
|------|----------|-------|--------|
| Daily Morning Briefing | 9 AM KST | DeepSeek | Running |
| Weekly Review | 10 PM Sunday KST | DeepSeek | Running |

### Known Issues

- Gmail OAuth tokens expired from March 31 — require re-authentication
- Heartbeat disabled (all recurring work scheduled via cron)

### Testing Artifacts

- Ollama logs: `/tmp/ollama.log`
- OpenClaw logs: `~/.openclaw/logs/`
- Model configs: `~/.openclaw/config.yaml` (local provider section)

---

## Appendix: Commands Reference

### Check Ollama Status

```bash
curl http://localhost:11434/api/tags
```

### Monitor Ollama Inference

```bash
tail -f /tmp/ollama.log
```

### Test Model Directly

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "gemma4:e4b",
  "prompt": "What is 2+2?",
  "stream": false
}'
```

### Unload Model from Memory

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "gemma4:e4b",
  "prompt": "",
  "keep_alive": 0
}'
```

### Check System Memory

```bash
top -l 1 | grep "PhysMem"
```

---

## Conclusion

The trial to integrate Gemma4 local inference into OpenClaw revealed both technical and capability-level constraints. While Ollama installation and basic inference proved reliable, Gemma4's instruction-tuned personality—designed for user-friendly, cautious behavior—is fundamentally misaligned with OpenClaw's requirement for autonomous tool execution. The 16GB Mac Mini system also demonstrated RAM constraints that limit practical deployment of larger models. The trial successfully documented correct configuration practices (native Ollama API, timeout tuning, dependency avoidance) and validated the original 24GB hardware recommendation. The recommendation to revert to cloud-based DeepSeek inference is based on capability requirements and economic analysis, with a pathway to revisit local inference if hardware and model capabilities improve in future iterations.
