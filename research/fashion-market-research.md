# Fashion Market Research Automation — TIME (타임) MD Use Case

**Research date:** 2026-04-01  
**Status:** Scoping — pending wife interview  
**Target:** Mac Mini M4, 16GB RAM — OpenClaw 2026.3.x

---

## What This Is

TIME (타임) is South Korea's #1 women's ready-to-wear brand by revenue, owned by 한섬 (THE HANDSOME, part of Hyundai Department Store Group). Known for clean premium design with no logo play, TIME just became the first Korean ready-to-wear brand to join the Paris Fashion Week official calendar (2026 F/W).

A **Merchandise Director (MD)** at TIME researches trends, sources inventory, and guides assortment planning. This use case automates the **competitive intelligence phase** by running an orchestrated multi-agent system that:

1. **Triggers on Telegram:** User sends "fashion research" or "패션 리서치"
2. **Spawns 8 parallel brand agents:** Each agent scrapes one brand's new arrivals page via Firecrawl
3. **Extracts structured data:** Top 5 items, Pantone color palette, styling observations, materials, and price architecture per brand
4. **Synthesizes results:** Merges data across all 8 brands, analyzes design gaps relative to TIME, adds strategic recommendations
5. **Delivers as Google Doc:** Uploads to Google Drive with public link, sends link via Telegram

The system tracks both Korean competitors (Recto, SYSTEM, SJSJ, Theory Korea) and global reference brands (The Row, Toteme, Lemaire, Max Mara) in a single run, completing in ~2–3 minutes with cost under $0.25.

---

## What the Agent Monitors

The OpenClaw agent should categorize and report on these dimensions each week:

### Color Trends
- Dominant palettes per season (primary 3–5 colors)
- Emerging colors breaking through (what's rising vs. fading)
- Color distribution by category: outerwear, knitwear, basics, dresses
- Regional variation: Korean vs. Western palettes

### Silhouette & Structure
- Fit profiles: oversized, fitted, hybrid cuts
- Length trends: midi, maxi, knee-length, mini (by category)
- Volume story: slim, straight, flared, bubble details
- Waist placement: natural, high, low
- Proportions: long-line blazers, cropped jackets, bias cuts

### Key Items / Categories (Top of Mind)
- Outerwear: trench, blazer, coat, leather jacket, denim jacket
- Bottoms: wide-leg trouser, straight leg, bias-cut pant, skirt types
- Knitwear: knit dress, knit cardigan, sweater
- Basics: white shirt, striped shirt, t-shirt profiles
- Dresses: shirt dress, wrap dress, maxi, slip dress

### Details & Embellishments
- Collar types: shirt collar, stand collar, unstructured, Mao
- Sleeve details: puffed, rolled, constructed cuff, button detail
- Closures: hidden buttons, asymmetric zips, traditional buttons
- Hardware: metal quality, logo placement, minimalist vs. visible
- Fabric texture: matte, sheen, twill, damask, technical

### Competitor Intelligence
- Korean premium brands: SYSTEM, MICHAA, SJSJ, Theory Korea, Weekend Max Mara Korea
- What are they stocking? Pricing? Fit philosophy? Color story?
- Where do they differentiate from TIME?

### Global Reference Brands
(Aesthetic territory — what TIME studies for design inspiration)
- Toteme — minimalist, understated luxury
- A.P.C. — French essentialist
- Officine Generale — architectural tailoring
- Cos — Scandinavian structure
- Arket — heritage basics
- The Row — refined simplicity

---

## Data Sources to Crawl

### 8-Brand Focus (Orchestrated Parallel Crawl)

The primary automation targets 8 specific brand sites — 4 Korean competitors and 4 global references. Each brand agent scrapes the new arrivals page.

| Brand | Category | URL to Scrape |
|-------|----------|---------------|
| **Recto** | Korean | https://en.recto.co.kr/collections/new-arrivals |
| **SYSTEM** | Korean | https://www.system.co.kr/new |
| **SJSJ** | Korean | https://www.sjsj.co.kr/category/new |
| **Theory Korea** | Korean | https://www.theory.co.kr/new-arrivals |
| **The Row** | Global | https://www.therow.com/en-us/new-arrivals |
| **Toteme** | Global | https://toteme-studio.com/collections/new-in |
| **Lemaire** | Global | https://www.lemaire.fr/en/new-arrivals |
| **Max Mara** | Global | https://www.maxmara.com/en/new-arrivals |

### Secondary Reference Sources

For optional manual review or future expansion:

| Source | Content | Value |
|--------|---------|-------|
| **Musinsa** | Korean multi-brand, 200+ local & global brands | Korean market gauge |
| **29cm** | Premium Korean multi-brand, strong local focus | Korean premium tracker |
| **W Concept** | Korean contemporary brands, clean aesthetic | Korean contemporary |
| **Net-a-Porter** | Premium ready-to-wear, fresh inventory weekly | Retail benchmark |
| **SSENSE** | Contemporary and designer, clean editorial | Retail + design reference |

**Primary focus:** 8-brand structured crawl runs on demand via Telegram trigger. Secondary sources can be added to monthly reports if MD requests deeper market mapping.

---

## Output Format (Finalized)

Report structure per brand, formatted as Markdown and uploaded to Google Drive as Google Doc:

### Per-Brand Template

```markdown
## [Brand Name]
🔗 [brand website]

### Top 5 New Arrivals
1. [Item Name](product link) — garment type — price
   ![product image](image url)
2. [Item Name](product link) — garment type — price
   ![product image](image url)

### Color Palette
| Swatch | Pantone | Hex | Name |
|--------|---------|-----|------|
| ■ | 13-0002 TCX | #F2EFE4 | Whitecap Gray |
| ■ | 14-1020 TCX | #2C2C2C | Almost Black |

### Styling Observations
- Description of how items are layered/photographed (proportions, fabric combinations, visual tone)
  ![styling image](url)
- Description of brand's garment construction philosophy (seams, hardware, finishing details)

### Material Keywords
List 5–8 keywords from product descriptions: e.g., "wool," "linen blend," "recycled nylon," "deadstock fabric," "minimal hardware," "hand-finished seams"

### Price Architecture
| Category | Range | Notes |
|----------|-------|-------|
| Outerwear | ₩3.5–6M / €2K–4K | Trench, blazer, coat |
| Suiting | ₩2.5–4.5M / €1.5K–3K | Jacket, trouser, dress |
| Knitwear | ₩1.5–3M / €800–1.8K | Cardigan, sweater, dress |
| Basics | ₩800K–1.8M / €400–1K | Shirt, tee |
```

### Report Synthesis

After all 8 agents complete, the orchestrator synthesizes a single report containing:

1. **All 8 brand sections** (per template above)
2. **Design Opportunities for TIME** — analysis of gaps and white space:
   - Which color palettes are underexplored?
   - Which silhouettes are trending that TIME isn't showing?
   - Where can TIME differentiate from competitors while staying true to brand?
   - Pricing gaps and positioning opportunities
3. **Metadata:** Report date, brands tracked, total items analyzed, cost per run

### Delivery

- Save to `~/.openclaw/workspace/fashion-research/[YYYY-MM-DD].md`
- Upload to Google Drive as Google Doc: `TIME Fashion Research — [date]`
- Set public link permission (anyone with link can view)
- Send Google Doc link via Telegram

---

## Delivery Flow

```
User (wife): Sends "fashion research" or "패션 리서치" via Telegram
    ↓
OpenClaw Orchestrator: Detects trigger keyword
    ↓
Spawn 8 parallel brand agents (one per brand)
    ├─ Agent 1: Scrape Recto new arrivals via Firecrawl → extract data → return structured result
    ├─ Agent 2: Scrape SYSTEM new arrivals via Firecrawl → extract data → return structured result
    ├─ Agent 3: Scrape SJSJ new arrivals via Firecrawl → extract data → return structured result
    ├─ Agent 4: Scrape Theory Korea new arrivals via Firecrawl → extract data → return structured result
    ├─ Agent 5: Scrape The Row new arrivals via Firecrawl → extract data → return structured result
    ├─ Agent 6: Scrape Toteme new arrivals via Firecrawl → extract data → return structured result
    ├─ Agent 7: Scrape Lemaire new arrivals via Firecrawl → extract data → return structured result
    └─ Agent 8: Scrape Max Mara new arrivals via Firecrawl → extract data → return structured result
    ↓
Synthesis Agent: Merge all 8 results into single report
    ├─ Format each brand per template
    ├─ Analyze "Design Opportunities for TIME"
    └─ Save to ~/.openclaw/workspace/fashion-research/[YYYY-MM-DD].md
    ↓
Google Drive Uploader:
    ├─ Upload Markdown file as Google Doc
    ├─ Rename: "TIME Fashion Research — [date]"
    └─ Set sharing: anyone with link, read-only
    ↓
Telegram Delivery:
    └─ Send message: "Fashion research complete. [Google Doc link]"
```

**Estimated runtime:** 2–3 minutes (8 parallel scrapes + synthesis)  
**Estimated cost per run:** $0.15–0.25 USD

---

## Questions for the Wife (MD Interview)

Before setting up automation, interview to clarify scope:

1. **Reference brands:** Which global brands do you treat as direct references when planning TIME collections? (Toteme, A.P.C., Officine Generale, The Row, others?)

2. **Weekly routine:** When you sit down to do trend research, what are you specifically looking for? Colors? Silhouettes? Item availability? Display/merchandising?

3. **Manual monitoring:** Which Korean fashion platforms do you already monitor (Musinsa, 29cm, W Concept, Instagram)? Which are most useful?

4. **Global vs. local:** Do you need global runway data (Vogue Runway, Paris shows) or is retail floor data (net-a-porter, SSENSE new arrivals) more actionable?

5. **Planning horizon:** What season are you typically planning for when you do trend research? (E.g., planning F/W 2026 now in April 2026, or already thinking about S/S 2027?)

6. **Competitor focus:** Are there specific Korean premium competitors (SYSTEM, MICHAA, SJSJ, Theory Korea) you want tracked closely? Any others?

7. **Category priority:** Is there a specific product category you want tracked more closely right now? (E.g., outerwear, knitwear, bottoms, details/accessories?)

8. **Buying cycles:** When do you typically visit shops or make sourcing decisions? (E.g., once a month, twice per season?) Does the report timing need to sync with those trips?

9. **Report format:** What's most useful — a weekly PDF digest, Telegram message with images, simple text + hex palette, or something else?

10. **Price sensitivity:** Should the report include price ranges? Is competitor pricing important for assortment decisions?

11. **Frequency:** Weekly on a specific day (Tuesday/Wednesday before meetings)? Or as-needed triggered reports?

12. **Details depth:** How much detail on fabric/texture? Construction details (collar types, cuff styles)? Hardware finishes?

13. **Brand story:** Are there regional fashion moments (Korean designers at international fashion weeks, local industry events) you want tracked?

14. **Existing tools:** What tools do you currently use for research? (Pinterest, WhatsApp collections, email archives, spreadsheets?) Any data you want to preserve or integrate?

15. **Success metric:** How would you know this automation is saving you time and improving buying decisions? What's the one metric?

---

## Suggested OpenClaw Setup

### Architecture

| Component | Choice | Reason |
|-----------|--------|--------|
| **Web crawling** | Firecrawl | Remote managed browser, handles JavaScript rendering and 403 blocks, parallel sessions |
| **Orchestrator model** | Claude Sonnet 4.6 | Spawn 8 parallel agents, coordinate timing, synthesis and strategic analysis |
| **Brand agent model** | DeepSeek V3.2 | Per-brand scraping, data extraction, structured output (~$0.28/$0.42 per 1M tokens) |
| **Synthesis model** | Claude Sonnet 4.6 | Merge results, analyze design opportunities, final report polish |
| **Google Drive CLI** | `gog drive` | Upload Markdown to Google Drive, convert to Doc, set sharing permissions |
| **Delivery channel** | Telegram | Link delivery to wife's phone |
| **Trigger type** | Telegram keyword | "fashion research" or "패션 리서치" in incoming message |

### Skill Installation

```bash
openclaw skill install firecrawl
# gog drive CLI already configured at ~/.openclaw/workspace
```

Verify:
```bash
openclaw skill list
```

### Model Routing (AGENTS.md entry, below)

```
Trigger: "fashion research" | "패션 리서치"
├─ Orchestrator: Claude Sonnet 4.6 (spawn 8 agents, coordinate)
├─ Brand agents (x8): DeepSeek V3.2 (scrape + extract)
├─ Synthesizer: Claude Sonnet 4.6 (merge + analyze)
└─ Delivery: gog drive + Telegram link
```

**Cost estimate per run:** $0.15–0.25 USD (8 parallel brand scrapes + synthesis)  
**Google Drive account:** thefightingbee@gmail.com (already configured)

---

## Implementation Roadmap

### Phase 1: Scoping (This week)
- [ ] Conduct wife interview (15 questions above)
- [ ] Document reference brands and data sources she uses
- [ ] Finalize output format preferences

### Phase 2: Setup (Week of 2026-04-07)
- [ ] Install Firecrawl skill
- [ ] Configure Telegram delivery
- [ ] Create HEARTBEAT.md with Tuesday cron
- [ ] Build initial data source list (URL index)

### Phase 3: Pilot (2026-04-14 first report)
- [ ] Manual crawl of 3–5 key sources
- [ ] Test color extraction + hex formatting
- [ ] Test report generation and Telegram delivery
- [ ] Adjust categories based on wife feedback

### Phase 4: Iterate (Ongoing)
- [ ] Week 2–4: Refine based on MD feedback
- [ ] Add/remove data sources
- [ ] Tune report sections (what's useful, what's noise)
- [ ] Consider paid forecasting services if value is clear

---

## Status

**Research date:** 2026-04-01  
**Status:** Implementation ready — pending AGENTS.md deployment  
**Multi-agent system spec:** Finalized (8 parallel brand agents, synthesis, Google Drive delivery)  
**Next action:** Deploy fashion-research agent definition in ~/.openclaw/workspace/AGENTS.md

---

## Glossary

| Term | Definition |
|------|------------|
| **MD (Merchandise Director)** | Fashion buyer role — researches trends, sources inventory, guides collection planning |
| **타임 (TIME)** | Korean premium women's ready-to-wear brand, owned by 한섬 |
| **한섬 (THE HANDSOME)** | Parent company, Hyundai Department Store Group subsidiary |
| **Ready-to-wear** | Off-the-rack clothing (vs. haute couture or bespoke tailoring) |
| **Silhouette** | Overall shape/proportion of a garment (oversized, fitted, trapeze, etc.) |
| **Assortment planning** | Process of deciding which items, colors, and quantities to stock |
| **Firecrawl** | Web scraping service with remote managed browser, integrated with OpenClaw |
| **Exa** | Semantic search API (optional for fashion queries) |
| **Hex code** | Color notation (e.g., #F5F5DC for beige) used for design reference |
| **SSENSE / Net-a-Porter** | Premium multi-brand retailers, design reference benchmarks |
| **Musinsa** | Korean multi-brand platform, aggregate of Korean and global fashion |

---

## Sources & References

- OpenClaw: [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)
- Firecrawl integration: [docs.firecrawl.dev/developer-guides/openclaw](https://docs.firecrawl.dev/developer-guides/openclaw)
- TIME (타임): [thehandsome.com](https://thehandsome.com)
- Paris Fashion Week 2026: [modeparis.com](https://modeparis.com)
