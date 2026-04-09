# Household Account Book — OpenClaw Integration Research

Research date: 2026-03-31
OpenClaw version target: 2026.3.x

---

## 1. Goal

Build a unified household finance view that aggregates transactions from **both spouses' bank accounts and credit cards** into a local database, with automated categorization, spending reports, and anomaly alerts — all delivered via Telegram through OpenClaw.

**Accounts to aggregate:**
- KB Bank (국민은행) — checking/savings
- Shinhan Bank (신한은행) — checking/savings
- Hyundai Card (현대카드) — credit card
- Additional cards as needed

---

## 2. The Korea Problem: Why Plaid Won't Work

Plaid, the standard US financial data aggregator, **does not support Korean banks**. Korea has its own financial data ecosystem:

| System | What it is | Personal dev access? |
|--------|-----------|---------------------|
| **오픈뱅킹 (Open Banking)** | KFTC-operated API for bank account queries across all Korean banks | **No** — requires 개인사업자 (sole proprietor registration) minimum. Not available to pure individuals. |
| **마이데이터 (MyData)** | Government-backed financial data aggregation (FSC licensed) | **No** — requires FSC (금융위원회) license. Only for licensed companies (69 providers as of 2023). |
| **Bank/Card Open APIs** | Individual bank developer portals (Shinhan, KB) | **Limited** — mostly for corporate partners, not personal use |

**Bottom line:** Korea's official APIs are designed for licensed fintech companies, not personal automation projects.

---

## 3. Practical Approaches (What Actually Works)

### Approach A: BankSalad Export (Recommended — Start Here)

[뱅크샐러드 (BankSalad)](https://www.banksalad.com/) is a licensed MyData provider that already aggregates all your banks and cards.

**How it works:**
1. Both you and wife install BankSalad, connect all accounts via MyData
2. BankSalad auto-categorizes all transactions (income/expense/transfer)
3. Export data: 가계부 → 설정 → 파일로 받기 → 데이터 내보내기
4. Receive Excel file with up to **1 year of transaction history**
5. OpenClaw ingests the Excel into local SQLite database

**Pros:**
- Zero scraping, zero API registration
- Covers ALL Korean banks and cards (KB, Shinhan, Hyundai Card, etc.)
- Already categorized by BankSalad's ML models
- Legal and ToS-compliant
- Includes 카카오페이, 토스 pay money transactions

**Cons:**
- Export is manual (must trigger from app, not automated)
- Maximum 1 year history per export
- Requires both spouses to use the app
- Export format may change without notice

**Automation workaround:** Set a monthly reminder via OpenClaw to prompt you to export. OpenClaw processes the file once you drop it in a watched folder. See Section 8 for full automation research.

### Approach B: Manual Bank/Card CSV Export

Each Korean bank and card company allows transaction history download from their website:

| Institution | Export path | Format |
|------------|------------|--------|
| KB국민은행 | 인터넷뱅킹 → 조회 → 거래내역조회 → 엑셀저장 | Excel/CSV |
| 신한은행 | SOL → 조회 → 계좌조회 → 거래내역 → 내려받기 | Excel/CSV |
| 현대카드 | My Account → 이용내역 → 엑셀 다운로드 | Excel |

**Pros:**
- Direct from source, most accurate
- No third-party app needed

**Cons:**
- Must log in to each bank/card separately (2FA each time)
- Very manual — 4+ exports per month for a household
- Different formats per institution

### Approach C: Python Screen Scraping (Advanced, Fragile)

Open-source tools exist but are limited:

| Tool | Banks supported | Status |
|------|----------------|--------|
| [`simple_bank_korea`](https://github.com/Beomi/simple_bank_korea) | KB국민은행 only | Requires '빠른조회' activation. Selenium-based. |
| [`kb_transaction`](https://github.com/Beomi/kb_transaction) | KB국민은행 only | Selenium + virtual keyboard handling |

**Not recommended** for production use:
- Breaks when bank UI changes (frequent in Korea)
- May trigger fraud alerts or account locks
- Violates most Korean banks' ToS
- No Shinhan or Hyundai Card support
- Requires maintaining browser automation infrastructure

### Approach D: SMS/Push Notification Parsing (Supplementary)

Korean banks send real-time transaction SMS notifications. These can be parsed for near-real-time tracking.

**How it works:**
- Korean banks send SMS for every transaction: `[현대카드] 승인 45,000원 스타벅스 03/31 14:23`
- Parse SMS on Android (via Tasker/MacroDroid → webhook to OpenClaw)
- iPhone: No direct SMS access. Could use email forwarding rules if bank supports email alerts.

**Pros:**
- Real-time transaction capture
- No bank login needed
- Works for all banks/cards

**Cons:**
- SMS parsing is brittle (format varies by institution)
- iPhone support is limited
- Supplementary only — doesn't replace full transaction history
- Misses details like running balance, merchant category

---

## 4. Recommended Architecture

```
Data Collection (monthly)
  ├─ BankSalad export (primary — both spouses)
  │   └─ Excel file dropped in ~/openclaw-data/imports/
  │
  ├─ SMS parsing (supplementary — real-time alerts)
  │   └─ Webhook → OpenClaw → SQLite
  │
  └─ Manual CSV (fallback for specific accounts)
      └─ Bank website → download → ~/openclaw-data/imports/

        │
        ▼

OpenClaw Ingestion Skill (custom)
  ├─ Parse Excel/CSV files (pandas)
  ├─ Normalize transaction format
  ├─ Deduplicate across sources
  ├─ LLM-enhanced categorization (refine BankSalad categories)
  └─ Insert into SQLite database

        │
        ▼

Local SQLite Database (~/openclaw-data/finance.db)
  ├─ transactions table
  ├─ categories table
  ├─ accounts table
  ├─ budgets table
  └─ monthly_summaries table (materialized)

        │
        ▼

OpenClaw Reporting (cron jobs)
  ├─ Monthly household report → Telegram
  ├─ Unusual charge alerts → Telegram
  └─ On-demand queries via Telegram ("이번 달 외식비 얼마?")
```

---

## 5. Database Schema

SQLite is the right choice for this use case — lightweight, zero-config, runs on Mac Mini, and OpenClaw's built-in `personal-finance` skill already uses it.

### Tables

```sql
-- Family members / account owners
CREATE TABLE members (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,           -- e.g., 'Gene', 'Wife'
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Bank accounts and credit cards
CREATE TABLE accounts (
    id INTEGER PRIMARY KEY,
    member_id INTEGER NOT NULL REFERENCES members(id),
    institution TEXT NOT NULL,    -- e.g., 'KB국민은행', '현대카드'
    account_type TEXT NOT NULL,   -- 'checking', 'savings', 'credit_card'
    account_name TEXT,            -- e.g., 'KB 주거래통장', '현대카드 M'
    last_four TEXT,               -- last 4 digits for identification
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Spending categories (hierarchical)
CREATE TABLE categories (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,           -- e.g., '식비', '교통', '의료'
    name_en TEXT,                 -- e.g., 'Food', 'Transport', 'Medical'
    parent_id INTEGER REFERENCES categories(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- All transactions
CREATE TABLE transactions (
    id INTEGER PRIMARY KEY,
    account_id INTEGER NOT NULL REFERENCES accounts(id),
    category_id INTEGER REFERENCES categories(id),
    transaction_date DATE NOT NULL,
    amount INTEGER NOT NULL,      -- in KRW (원), negative = expense
    description TEXT,             -- merchant name / memo
    merchant_name TEXT,           -- cleaned merchant name
    transaction_type TEXT NOT NULL, -- 'income', 'expense', 'transfer'
    source TEXT,                  -- 'banksalad', 'manual_csv', 'sms'
    source_file TEXT,             -- original import filename
    raw_data TEXT,                -- original row as JSON for debugging
    is_duplicate BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(account_id, transaction_date, amount, description) -- dedup key
);

-- Monthly budgets per category
CREATE TABLE budgets (
    id INTEGER PRIMARY KEY,
    category_id INTEGER NOT NULL REFERENCES categories(id),
    month TEXT NOT NULL,          -- e.g., '2026-03'
    budget_amount INTEGER NOT NULL, -- in KRW
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(category_id, month)
);
```

### Default Categories

```sql
INSERT INTO categories (name, name_en) VALUES
('식비', 'Food & Dining'),
('교통', 'Transportation'),
('주거', 'Housing & Rent'),
('공과금', 'Utilities'),
('통신', 'Phone & Internet'),
('의료', 'Medical & Health'),
('교육', 'Education'),
('쇼핑', 'Shopping'),
('의류', 'Clothing'),
('문화/여가', 'Entertainment'),
('여행', 'Travel'),
('구독서비스', 'Subscriptions'),
('보험', 'Insurance'),
('경조사', 'Gifts & Events'),
('반려동물', 'Pets'),
('기타', 'Miscellaneous');

-- Sub-categories (examples)
INSERT INTO categories (name, name_en, parent_id) VALUES
('외식', 'Dining Out', 1),
('식료품', 'Groceries', 1),
('배달', 'Delivery', 1),
('카페', 'Coffee & Cafe', 1),
('대중교통', 'Public Transit', 2),
('택시', 'Taxi', 2),
('주유', 'Gas', 2),
('주차', 'Parking', 2);
```

---

## 6. Transaction Categorization Strategy

### Layer 1: BankSalad Categories (Free)

BankSalad already categorizes transactions via its MyData integration. The Excel export includes category columns. Map these to our schema on import.

### Layer 2: LLM-Enhanced Categorization (OpenClaw)

For transactions that BankSalad miscategorizes or doesn't categorize, use an LLM:

```
Prompt: "Categorize this Korean transaction into one of these categories: 
[식비, 교통, 주거, 공과금, 통신, 의료, 교육, 쇼핑, 의류, 문화/여가, 여행, 구독서비스, 보험, 경조사, 기타]

Transaction: '스타벅스 강남점 15,000원'
→ 식비 > 카페

Transaction: '쿠팡 배달 32,000원'  
→ 식비 > 배달"
```

Use **DeepSeek V3.2** for bulk categorization (cheapest) and **Claude** only for ambiguous cases.

### Layer 3: User Corrections (Learning)

Store user corrections and build a merchant→category mapping table:

```sql
CREATE TABLE merchant_rules (
    id INTEGER PRIMARY KEY,
    merchant_pattern TEXT NOT NULL,  -- regex or substring match
    category_id INTEGER NOT NULL REFERENCES categories(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Example rules
INSERT INTO merchant_rules (merchant_pattern, category_id) VALUES
('스타벅스%', 1),     -- 식비 > 카페
('쿠팡%', 8),         -- 쇼핑
('배달의민족%', 1),   -- 식비 > 배달
('카카오택시%', 2);   -- 교통 > 택시
```

---

## 7. OpenClaw Integration

### Install the personal-finance skill

```bash
openclaw skills install personal-finance
```

This gives a base SQLite setup. We'll extend it with a custom skill for Korean bank data.

### Custom skill: `household-finance-kr`

Build a custom OpenClaw skill that:
1. Watches `~/openclaw-data/imports/` for new Excel/CSV files
2. Parses BankSalad export format (or raw bank CSV)
3. Normalizes and deduplicates transactions
4. Categorizes via merchant rules → LLM fallback
5. Inserts into `finance.db`

### Cron jobs

```bash
# Monthly household report — 1st of each month at 9 AM
openclaw cron add \
  --name "monthly-household-report" \
  --cron "0 9 1 * *" \
  --tz "Asia/Seoul" \
  --session isolated \
  --message "Generate the monthly household finance report from finance.db. Include: total income vs expenses, spending by category (top 10), comparison vs last month, unusual charges (>200,000원 single transactions), and budget vs actual for any set budgets. Format as a clean report with Korean won amounts." \
  --model sonnet \
  --announce \
  --channel telegram

# Weekly spending snapshot — every Sunday at 10 AM
openclaw cron add \
  --name "weekly-spending-snapshot" \
  --cron "0 10 * * 0" \
  --tz "Asia/Seoul" \
  --session isolated \
  --message "Generate a quick weekly spending snapshot from finance.db. Show this week's total spending by category, highlight the top 5 merchants, and flag any transactions over 100,000원. Keep it brief." \
  --model haiku \
  --announce \
  --channel telegram

# Unusual charge alert — check daily at 8 PM
openclaw cron add \
  --name "unusual-charge-alert" \
  --cron "0 20 * * *" \
  --tz "Asia/Seoul" \
  --session isolated \
  --message "Check finance.db for any transactions today over 200,000원 or any new recurring charges not seen before. If found, alert with details. If nothing unusual, do not send a message." \
  --model haiku \
  --announce \
  --channel telegram
```

### On-demand queries via Telegram

Once the database is populated, ask questions naturally:

- "이번 달 외식비 얼마 썼어?" → queries transactions where category = 식비/외식
- "지난 3개월 구독서비스 목록 보여줘" → lists recurring subscription charges
- "아내 현대카드 이번 달 사용액은?" → filters by member + account
- "작년 대비 교통비 변화 알려줘" → year-over-year comparison

---

## 8. Export Automation Research (Decision Pending)

The BankSalad export is manual (3 taps in the app). We researched whether OpenClaw can automate this entirely.

### Option 1: iPhone Mirroring + mirroir-mcp (Investigated)

**Stack:** iPhone → macOS iPhone Mirroring → [mirroir-mcp](https://github.com/jfarcand/iphone-mirroir-mcp) (MCP server) → OpenClaw

**How it works:**
- `mirroir-mcp` is an MCP server that controls a real iPhone through macOS iPhone Mirroring
- Uses Apple Vision OCR to find UI elements — no hardcoded coordinates
- Supports tap, swipe, type, screenshot
- Can create reusable Skills (step-by-step workflows) for repeatable automation
- Install: `brew tap jfarcand/tap && brew install mirroir-mcp`

**Requirements:** macOS 15 (Sequoia)+, Screen Recording + Accessibility permissions

**Verdict: Won't work unattended.**
- iPhone Mirroring requires the phone to be **unlocked and nearby** (same Wi-Fi/Bluetooth)
- BankSalad requires **Face ID / biometric auth** on app open
- If you leave with your phone, the session drops
- Not suitable for fully automated monthly cron jobs

### Option 2: OpenClaw Peekaboo + Desktop Automation

[Peekaboo](https://github.com/steipete/Peekaboo) is OpenClaw's macOS UI automation skill — screenshot, click, type, drag, scroll, hotkey, app/window management.

- Install: `brew install steipete/tap/peekaboo`
- Could automate Mac-side tasks (processing files, opening apps) but **can't reach iPhone apps**

### Option 3: Android Emulator on Mac Mini (Not Yet Explored)

Run BankSalad in an Android emulator (LD Player, BlueStacks) on the Mac Mini. OpenClaw automates via Peekaboo screen control. Fully unattended.

**Unknowns:**
- BankSalad may require phone SMS verification for login
- Android emulator performance on Mac Mini M4
- Whether BankSalad blocks emulator environments

### Option 4: Monthly Reminder + Manual Export (Simplest)

OpenClaw sends a Telegram reminder on the 1st of each month. You do the 3 taps in BankSalad, file lands in a watched folder, OpenClaw auto-ingests.

**Effort: 10 seconds/month.**

### Current Decision: TBD

Will decide between these options during implementation. Option 4 (manual + reminder) is the pragmatic starting point. Options 1-3 can be revisited once the core pipeline (ingestion → database → reports) is working.

---

## 9. Implementation Roadmap

> **Note:** Export automation (Section 8) is deferred. Start with manual export + auto-ingestion.

| Step | Task | Effort |
|------|------|--------|
| 1 | Both install BankSalad, connect all accounts via MyData | 30 min |
| 2 | Export first BankSalad Excel, analyze format | 15 min |
| 3 | Create SQLite schema (`finance.db`) on Mac Mini | 30 min |
| 4 | Build Excel parser script (Python/pandas) | 2-3 hours |
| 5 | Install OpenClaw `personal-finance` skill, extend for Korean data | 1-2 hours |
| 6 | Set up import folder watcher | 1 hour |
| 7 | Configure cron jobs (monthly report, weekly snapshot, alerts) | 30 min |
| 8 | Test with 1 month of real data | 1 hour |
| 9 | Add LLM categorization for uncategorized transactions | 1-2 hours |
| 10 | Add merchant_rules for frequent merchants | Ongoing |

**Total estimated setup: ~1 day of focused work**

---

## 10. Cost Estimate

| Component | Cost |
|-----------|------|
| BankSalad app | Free |
| SQLite | Free |
| OpenClaw personal-finance skill | Free |
| LLM categorization (DeepSeek, ~500 tx/month) | < $0.10/month |
| Monthly report generation (Sonnet) | ~$0.05/report |
| Weekly snapshots (Haiku) | ~$0.01/week |
| **Total** | **< $1/month** |

---

## 11. Future Enhancements

| Enhancement | When |
|-------------|------|
| SMS transaction parsing for real-time alerts | After core is stable |
| Automated BankSalad export (see Section 8) | After core pipeline is stable |
| Budget tracking with progress alerts ("식비 80% 소진") | Month 2 |
| Year-end tax summary (연말정산 prep) | December |
| Investment account integration (증권계좌) | After UC-4 |
| Shared family dashboard (simple web UI on Mac Mini) | Nice-to-have |

---

## 12. Open Questions

| Item | Decision needed |
|------|----------------|
| BankSalad vs Toss | Both are MyData providers. Need to check which export format is cleaner. Try both? |
| Wife's participation | She needs to install BankSalad and export monthly. Get buy-in. |
| Category granularity | Start with 16 top-level categories? Or go deeper from day 1? |
| Historical data | How far back do you want to go? BankSalad exports 1 year max. |
| Currency | Any USD/foreign currency transactions to handle? |

---

## 13. Sources

- [KFTC Open Banking Developer Portal](https://developers.kftc.or.kr/dev)
- [KFTC Open Banking Service Portal](https://openapi.kftc.or.kr)
- [MyData Korea Developer Testbed](https://developers.mydatakorea.org/)
- [Shinhan Open API](https://openapi.shinhan.com/)
- [KB Group Open API Portal](https://apiportal.kbfg.com/intro)
- [BankSalad](https://www.banksalad.com/)
- [simple_bank_korea (GitHub)](https://github.com/Beomi/simple_bank_korea)
- [kb_transaction (GitHub)](https://github.com/Beomi/kb_transaction)
- [OpenClaw personal-finance skill](https://playbooks.com/skills/openclaw/skills/personal-finance)
- [MyData 2.0 — Development Asia](https://development.asia/insight/transforming-republic-koreas-banking-industry-through-mydata-20)
- [mirroir-mcp — iPhone control via macOS Mirroring](https://github.com/jfarcand/iphone-mirroir-mcp)
- [Peekaboo — macOS UI automation](https://github.com/steipete/Peekaboo)
- [OpenClaw Browser Automation Docs](https://docs.openclaw.ai/tools/browser)
