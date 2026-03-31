# 03 — Multi-Agent Setup

> **Source notes:** Based on workspace research notes dated 2026-03-30.
> Live docs: [docs.openclaw.ai/multi-agent](https://docs.openclaw.ai/multi-agent) [unverified — not fetched this session]

<!-- IMAGE: Multi-agent pipeline architecture diagram showing orchestrator and sub-agents -->

---

## How Multi-Agent Pipelines Work in OpenClaw

OpenClaw's multi-agent system runs parallel sub-agents, each scoped to a specific task with its own model assignment and workspace. An orchestrator agent coordinates them, routes data, and synthesizes outputs.

### Core concepts:

| Concept | Description |
|---------|-------------|
| **Orchestrator** | The primary agent that breaks down tasks, assigns to sub-agents, and compiles results |
| **Sub-agent** | A scoped agent with a specific task, model, and optional workspace files |
| **Pipeline** | A defined sequence/parallel set of sub-agent executions |
| **Handoff** | Structured data passing between sub-agents (usually markdown files) |
| **Model routing** | Each sub-agent can be assigned a different model for cost/capability optimization |

---

## Sub-Agent Model Assignments

Each agent in a pipeline should be assigned the most appropriate (cost-efficient) model for its task:

| Agent Role | Recommended Model | Rationale |
|-----------|-------------------|-----------|
| Orchestrator | `openrouter/anthropic/claude-sonnet-4-6` | Best reasoning for task decomposition and synthesis |
| Web data fetcher | `openrouter/deepseek/deepseek-v3.2` | Cheapest — handles bulk scraping + summarization |
| Search researcher | `openrouter/openai/gpt-4o` | Built-in web search capability |
| Data/metrics analyst | `openrouter/google/gemini-2.5-pro` | Strong structured extraction at mid cost |
| Risk assessor | `openrouter/google/gemini-2.5-pro` | Same: analysis at mid cost |
| Report writer / thesis | `openrouter/anthropic/claude-sonnet-4-6` | Final synthesis requires strongest reasoning |

---

## AGENTS.md — Pipeline Definitions

Define your multi-agent workflows in `~/.openclaw/workspace/AGENTS.md`. This file is the core of your pipeline configuration. [unverified: exact AGENTS.md schema — structure below is based on patterns observed in community examples]

```markdown
# Agents

## orchestrator
model: openrouter/anthropic/claude-sonnet-4-6
role: >
  You are the orchestrator. When the user sends a research request,
  decompose it into subtasks, dispatch to sub-agents, collect outputs,
  and compile the final report.

## data-fetcher
model: openrouter/deepseek/deepseek-v3.2
role: >
  You are the data fetcher. Use Firecrawl to scrape target URLs.
  Output raw markdown files to the shared workspace.
  Do not summarize — preserve all data for downstream agents.

## metrics-analyst
model: openrouter/google/gemini-2.5-pro
role: >
  You are the metrics analyst. Read scraped data files from the workspace.
  Extract financial metrics: revenue, margins, growth rates, valuations.
  Output a structured markdown table.

## risk-assessor
model: openrouter/google/gemini-2.5-pro
role: >
  You are the risk assessor. Read all analyst outputs.
  Identify key risks: macro, sector, company-specific.
  Output a structured risk register with severity ratings.

## thesis-writer
model: openrouter/anthropic/claude-sonnet-4-6
role: >
  You are the thesis writer. Read all upstream outputs.
  Synthesize a concise investment thesis with:
  - Bull case, bear case, base case
  - Key metrics table
  - Risk summary
  - Recommended action
```

---

## Orchestrator Patterns

### Pattern 1 — Sequential Pipeline

Tasks run in order, each agent consuming the previous agent's output:

```
[user request]
    → orchestrator (scope + plan)
    → data-fetcher (scrape)
    → metrics-analyst (extract)
    → risk-assessor (assess)
    → thesis-writer (synthesize)
    → orchestrator (compile + deliver)
```

**When to use:** When each stage depends on the previous (most financial research pipelines).

### Pattern 2 — Parallel Fan-Out

Multiple agents run simultaneously, orchestrator merges results:

```
[user request]
    → orchestrator (decompose)
    ├── data-fetcher-1 (source A)  ─┐
    ├── data-fetcher-2 (source B)   ├── orchestrator (merge)
    └── data-fetcher-3 (source C)  ─┘
    → thesis-writer (synthesize merged data)
```

**When to use:** When gathering data from independent sources simultaneously. Reduces total pipeline time significantly.

### Pattern 3 — Hybrid (Recommended for Financial Research)

```
[user request]
    → orchestrator (scope)
    │
    ├── [PARALLEL]
    │   ├── data-fetcher-A (company filings)
    │   ├── data-fetcher-B (news/sentiment)
    │   └── data-fetcher-C (competitor data)
    │
    → [SEQUENTIAL]
    → metrics-analyst (unified analysis)
    → risk-assessor
    → thesis-writer
    → orchestrator (final report → Telegram)
```

---

## Financial Research Pipeline — Full Example

This is the recommended architecture for the Mac Mini M4 financial research use case.

<!-- IMAGE: Financial research pipeline flowchart with model labels at each stage -->

### Pipeline Flow

```
User sends via Telegram:
  "research AAPL Q1 2026 earnings"
        │
        ▼
Claude Sonnet (Orchestrator)
  - Parses intent
  - Defines 4 subtasks
  - Dispatches to sub-agents
        │
        ├─────────────────────────────────────────────────┐
        ▼                                                   ▼
DeepSeek V3.2 (Data Fetcher A)              DeepSeek V3.2 (Data Fetcher B)
  Firecrawl → SEC filings                     Firecrawl → news + analyst reports
  Output: raw_filings.md                       Output: raw_news.md
        │                                                   │
        └──────────────────┬────────────────────────────────┘
                           ▼
              Gemini 2.5 Pro (Metrics Analyst)
                Reads: raw_filings.md + raw_news.md
                Extracts: revenue, EPS, guidance, margins
                Output: metrics_table.md
                           │
                           ▼
              Gemini 2.5 Pro (Risk Assessor)
                Reads: metrics_table.md + raw_news.md
                Identifies: macro risks, company risks, sector risks
                Output: risk_register.md
                           │
                           ▼
              Claude Sonnet (Thesis Writer)
                Reads: all upstream outputs
                Writes: investment_thesis.md
                           │
                           ▼
              Claude Sonnet (Orchestrator)
                Compiles master report
                Formats for Telegram
                           │
                           ▼
              Delivered via Telegram bot
```

### Estimated Cost per Research Run

| Stage | Model | Tokens (estimated) | Cost (estimated) |
|-------|-------|-------------------|-----------------|
| Orchestration (x2) | Claude Sonnet 4.6 | ~10k | ~$0.18 |
| Data fetching (x2) | DeepSeek V3.2 | ~50k | ~$0.04 |
| Metrics analysis | Gemini 2.5 Pro | ~20k | ~$0.13 |
| Risk assessment | Gemini 2.5 Pro | ~15k | ~$0.09 |
| Thesis writing | Claude Sonnet 4.6 | ~15k | ~$0.28 |
| **Total per run** | — | ~110k | **~$0.72** |

[All cost figures are estimated based on published per-token rates as of Q1 2026]

---

## Parallel Execution Notes

OpenClaw supports parallel sub-agent execution [unverified: exact concurrency model — based on community documentation patterns]:

- Sub-agents run as separate Node.js worker threads or child processes
- Shared workspace directory allows file-based data passing between agents
- The orchestrator polls for output files or receives callbacks on completion
- Recommended max concurrent agents on Mac Mini M4 16GB: **4–5 simultaneous**

### RAM usage estimate for 4 concurrent agents:

| Component | RAM |
|-----------|-----|
| macOS idle | ~2–3 GB |
| OpenClaw daemon | ~300 MB |
| 4 sub-agents (no browser) | ~2–4 GB |
| Firecrawl (remote, no local RAM) | 0 |
| **Total** | **~5–8 GB** |

16GB RAM is comfortable for 4–5 concurrent agents when using Firecrawl (remote browser — no local Chromium).

---

## Triggering Pipelines via Telegram

Once configured, pipelines can be triggered by natural language commands via Telegram:

```
"research TSLA 2026 outlook"
→ triggers full research pipeline

"brief me on today's market"
→ triggers quick morning briefing pipeline

"compare AAPL vs MSFT margins"
→ triggers comparative analysis pipeline
```

The orchestrator in AGENTS.md should define intent patterns for each pipeline trigger.

---

## Sources

- Workspace notes: `research/setup-guide-2026-03-30.md`
- Workspace notes: `research/mac-mini-specs-2026-03-30.md`
- [docs.openclaw.ai/multi-agent](https://docs.openclaw.ai/multi-agent) [link unverified this session]
- [github.com/digitalknk/openclaw-runbook](https://github.com/digitalknk/openclaw-runbook) [link unverified this session]
