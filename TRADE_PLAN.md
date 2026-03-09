# Trading App — Full Integration Plan
> Created: 2026-03-06
> Based on: 1-hour budget app analysis + financial data review

---

## Part 1: What the Budget App Tells Us (Foundation)

### What the budget app currently knows about you

From 29 months of real transaction data:

| Metric | Value |
|--------|-------|
| Combined take-home | ~$9,200/mo |
| What you actually keep | ~$297/mo |
| Credit card interest/mo | ~$68/mo |
| Eating out/mo | ~$953/mo |
| Amazon/mo | ~$944/mo |
| Robinhood transfers (29 mo) | $13,163 total (~$454/mo) |
| Savings transfers | $7,500 total |
| Subscriptions/mo | ~$86/mo |

### The hard truth before trading

Before committing capital to trading, the budget data forces this question:

> "Do you have $X that, if lost entirely, would not cause an overdraft, missed payment, or new credit card balance?"

Right now, based on your data:
- You have ~$30k+ in credit card debt cycling
- Two months this year you spent MORE than you earned
- You overdrafted 3x
- Your real monthly surplus is ~$297

**This means:** Your trading capital right now is whatever you can carve out from the Robinhood habit ($454/mo) without increasing debt. NOT a lump sum. NOT borrowed capital. NOT money you'd need in 3 months.

**The goal of this integration is to make that number precise, automatic, and hard to cheat.**

---

## Part 2: Budget App Improvements Required First

Before the trading app is useful, the budget app needs a data export layer. These are the only additions needed — no visual redesign.

### 2.1 — Add a Financial Snapshot Export

**Where:** Add an "Export for Trading" button to `index.html` (below the summary cards)

**What it produces:** `financial_snapshot.json` — a small file the trading app reads

```json
{
  "snapshot_date": "2026-03-06",
  "monthly_takehome": 9200,
  "fixed_expenses": {
    "mortgage_rent": 0,
    "utilities": 0,
    "insurance": 238,
    "subscriptions": 86,
    "debt_payments": 410
  },
  "variable_avg": {
    "food_restaurants": 953,
    "groceries": 472,
    "amazon_shopping": 944,
    "gas": 120,
    "other": 400
  },
  "actual_surplus": 297,
  "trading_allocation_pct": 10,
  "trading_capital_available": 29.70,
  "monthly_investment_habit": 454,
  "emergency_fund_months": 0,
  "debt_flag": true,
  "overdraft_flag": true
}
```

**Why this matters:** The trading app reads this file on load. It knows your financial position before you make any trade decision. If `debt_flag: true`, it shows a warning. If `emergency_fund_months: 0`, it warns you before any trade over $100.

### 2.2 — Add a Surplus Calculator Panel

Add a collapsible section to `index.html` after the summary cards:

```
TRADEABLE SURPLUS CALCULATOR
─────────────────────────────
Take-home:              $9,200
- Fixed costs:          -$734
- Variable avg:         -$2,889
- Debt minimum:         -$410
- Savings target:       -$500 (user-set)
═══════════════════════════════
True surplus:           $4,667
Trading allocation:     10%   [slider]
Monthly trading budget: $466  ← this feeds the trading app
```

**User controls:**
- Savings target (slider: $0–$2,000)
- Trading allocation % (slider: 0–25%)
- Lock button: "I commit to this allocation for this month"

### 2.3 — Shared localStorage Bridge

Both apps live locally. They share state via localStorage using a single key:

```javascript
// Budget app writes:
localStorage.setItem('romeo_financial_state', JSON.stringify({
  tradingCapital: 466,
  riskFlag: true,         // debt or no emergency fund
  lastUpdated: '2026-03-06',
  monthlyHabit: 454,
  allocationPct: 10
}));

// Trading app reads:
const fin = JSON.parse(localStorage.getItem('romeo_financial_state') || '{}');
```

This is the zero-dependency integration. No server, no API, no file imports. Just open both apps in the same browser.

---

## Part 3: The Trading App — Architecture

### Core Modules (in build order)

```
trading-app/
├── index.html         ← Main app (currently building)
│   ├── Module 1: Capital Gate
│   ├── Module 2: Rules Engine
│   ├── Module 3: Trade Advisor
│   └── Module 4: Trade Journal
├── CLAUDE.md
├── TRADE_PLAN.md      ← this file
└── README.md
```

---

## Part 4: Module 1 — Capital Gate (Build Phase 1)

**Question it answers:** "How much money can I trade right now?"

### UI Sections:

**A. Financial Status Banner** (reads from localStorage bridge)
```
[ FROM BUDGET APP ]  Last synced: 2026-03-06
Monthly trading budget:   $466
Available capital (est):  $1,200   ← user enters Robinhood balance
⚠ Risk flags active:  Debt detected · No emergency fund
```

**B. Risk Profile**
Three modes the user selects:
- `CONSERVATIVE` — max 2% per trade, max 5 open positions
- `MODERATE` — max 5% per trade, max 8 open positions
- `AGGRESSIVE` — max 10% per trade, max 12 open positions (shows warning if debt active)

**C. Capital Breakdown**
```
Total available:        $1,200
Per-trade max (5%):      $60
Max open positions:        8
Monthly add:             $466
Currently deployed:       $0   ← user enters
Available for trades:  $1,200
```

**Hard rules enforced in UI:**
- If `emergency_fund_months == 0` → never suggest trades over $200
- If `debt_flag == true` → show persistent warning "Pay down debt first"
- If monthly surplus < $100 → disable aggressive mode
- Never suggest using more than 20% of capital on a single trade, regardless of profile

---

## Part 5: Module 2 — News Rules Engine (Build Phase 2)

**Question it answers:** "What conditions should trigger me to act?"

This is the core differentiator. You set your own rules — the app enforces them so you don't trade emotionally.

### Rule Schema

Each rule has:
```javascript
{
  id: 'rule_001',
  name: 'NVDA earnings dip',
  ticker: 'NVDA',
  type: 'PRICE_DROP',         // or: PRICE_RISE, EARNINGS, SECTOR_NEWS, MACRO
  trigger_pct: -5,            // -5% drop triggers this
  action: 'BUY_SIGNAL',       // or: SELL_SIGNAL, WATCH, HOLD
  position_pct: 5,            // use 5% of capital if triggered
  stop_loss_pct: -8,          // exit if it drops another 8%
  take_profit_pct: 15,        // exit if up 15%
  notes: 'NVDA always recovers within 2 weeks after a 5% earnings dip',
  active: true,
  created: '2026-03-06'
}
```

### Rule Types Available

| Type | What triggers it | Example |
|------|-----------------|---------|
| PRICE_DROP | Stock drops X% in one day | NVDA drops 5% → buy signal |
| PRICE_RISE | Stock rises X% | SPY up 2% → take profit signal |
| EARNINGS | Earnings release day | AAPL earnings day → watch signal |
| SECTOR_NEWS | News keyword match | "Fed rate cut" → buy signal |
| MACRO | Market-wide event | VIX spikes over 30 → reduce position |
| MANUAL | User-triggered | I did research, I see an opportunity |

### Rule Builder UI

A simple form with:
- Ticker input (or "MARKET" for indices, "SECTOR:TECH" etc.)
- Trigger type dropdown
- Trigger threshold (number + %)
- Action dropdown
- Position size (% of capital, capped by risk profile)
- Stop loss %
- Take profit %
- Notes (freetext — this is your "why")
- Active toggle

**Rule Library:** All saved rules shown as cards. Toggle on/off. Edit/delete. Sort by ticker.

### News Simulation (Phase 2, no API yet)

In Phase 2, news rules are manually evaluated. You read news yourself, then:
1. Open the trading app
2. Go to "Check Rules"
3. Enter: "NVDA is down 5.2% today"
4. The app shows all rules that would trigger on this event
5. You confirm or skip

Phase 4 (Level 3 upgrade) adds real news feed ingestion.

---

## Part 6: Module 3 — Trade Advisor (Build Phase 3)

**Question it answers:** "Given my capital and active rules, what specific trade should I make?"

### Watchlist

Ticker cards showing:
- Last manually-entered price
- Rules active for this ticker
- Current signal status (WATCH / BUY / SELL / HOLD / NONE)
- Entry price (if you're in a trade)
- P&L (manual calculation)

### Signal Dashboard

When you open the app, you see all currently triggered rules:

```
ACTIVE SIGNALS  ●  2026-03-06

[ BUY SIGNAL ]  NVDA
Rule: "NVDA earnings dip"
Triggered: Price down 5.2% (threshold: -5%)
Suggested position: $60 (5% of $1,200)
Stop loss at: $115 (-8% from $125 entry)
Take profit at: $144 (+15%)
Notes: "NVDA always recovers..."

[ WATCH ]  AAPL
Rule: "AAPL earnings day"
Triggered: Earnings report today
Action: Watch — wait for reaction
```

### Position Sizer

Given a specific trade idea, the sizer outputs:

```
POSITION SIZER
──────────────────────────
Ticker:          NVDA
Current price:   $125
Position %:      5% of $1,200
Dollar amount:   $60
Shares to buy:   0.48  (fractional ok in Robinhood)

Stop loss:       $115  (8% below entry = lose $4.80)
Take profit:     $143.75  (15% above entry = gain $9)

Risk/Reward:     1:1.87
Max loss:        $4.80  (0.4% of total capital)

VERDICT: ✓ This trade fits your risk profile.
```

---

## Part 7: Module 4 — Trade Journal (Build Phase 4)

**Question it answers:** "Am I actually making money? What's working?"

Every completed trade is logged:

```javascript
{
  id: 'trade_001',
  ticker: 'NVDA',
  rule_triggered: 'rule_001',
  entry_price: 125,
  entry_date: '2026-03-06',
  shares: 0.48,
  cost_basis: 60,
  exit_price: 143.75,
  exit_date: '2026-03-20',
  exit_reason: 'take_profit',
  pnl: 9.00,
  pnl_pct: 15,
  notes: 'Went exactly as planned'
}
```

**Journal views:**
- All trades list (sortable)
- Win rate by rule (which rules actually work?)
- Win rate by ticker
- Monthly P&L summary
- Best/worst trades
- Capital growth curve (starting capital + all P&L over time)

**Rule performance view:**
```
RULE PERFORMANCE
────────────────────────────────
Rule                  Trades  Win%   Avg P&L
NVDA earnings dip       6     83%    +$8.20
AAPL pre-earnings       3     33%    -$2.10
SPY dip buy             4     75%    +$5.50
```

This is how you improve your rules over time — with real data.

---

## Part 8: Phase 5 — Level 3 Upgrade (Intelligence Layer)

Once the Level 1 app is working and you've logged 20+ trades, upgrade to Level 3:

### Backend additions (Flask/Python):

**8.1 News Feed Ingestion**
- RSS feeds: CNBC, Bloomberg, Reuters, SEC filings
- Keyword extraction per ticker in your watchlist
- Auto-flag news that might trigger your rules
- Store in SQLite

**8.2 Claude API Integration**
```python
# For each relevant news item:
prompt = f"""
Given this news: "{news_text}"
And this trading rule: {rule}
My current financial state: {fin_state}
Should I act on this? What's the risk?
"""
response = anthropic.messages.create(...)
```

**8.3 Automated Rule Checking**
- Every hour, check if any rules are triggered by latest prices (Yahoo Finance API, free)
- Push alerts to browser notification (or Slack via MCP)
- No auto-trading — always requires your confirmation

**8.4 Monarch Money → Auto-sync**
- Replace CSV import with Monarch agent (already in dev at `~/dev/Playground/experiment/`)
- Budget snapshot auto-updates daily
- Trading capital re-calculated each morning

---

## Part 9: The Full Integration Flow (End State)

```
MORNING ROUTINE
───────────────
1. Budget app auto-syncs from Monarch → updates financial_snapshot.json
2. Trading app reads snapshot → recalculates available capital
3. News feed runs → Claude evaluates news against your rules
4. Trading app shows: "2 signals active today. $466 available this month."
5. You open trading app → see actionable signals with sizing
6. You make trade in Robinhood → log it in trading app in 30 seconds
7. End of month: budget app shows you saved X% more, trading app shows P&L

EVERY DECISION ROOTED IN:
- Budget data (what you can actually afford)
- Your own rules (no impulse trading)
- Risk profile (no oversizing)
- Real P&L history (no rose-colored glasses)
```

---

## Part 10: Build Roadmap (Time Estimates)

| Phase | Description | Time | Priority |
|-------|-------------|------|----------|
| **0** | Budget app: surplus calc + export | 1 session | Before trading |
| **1** | Trading app: Capital Gate module | 1 session | Start here |
| **2** | Rules Engine: builder + library | 1-2 sessions | Core feature |
| **3** | Trade Advisor: signals + position sizer | 1-2 sessions | Core feature |
| **4** | Trade Journal: logging + performance | 1 session | After 10 trades |
| **5** | Level 3 upgrade: news feed + Claude API | 3-4 sessions | After consistent use |
| **6** | Monarch auto-sync integration | 1 session | Phase 5 prereq |

**Total to useful MVP:** ~4 sessions (Phases 0–3)
**Total to full system:** ~10-12 sessions (all phases)

---

## Part 11: Warning / Financial Reality Check

The budget app data shows clearly:
- You have **$30k+ in credit card debt** at ~20% APR
- You pay ~$68/mo in credit card interest (money burned)
- Every $1,000 of debt costs you ~$200/year

**Mathematically:**
Paying off $1,000 of 20% APR debt = guaranteed 20% return.
Trading to beat 20% returns is hard and risky.

**Recommended allocation order:**
1. Zero out overdraft buffer first ($1,000 in checking) — saves $34/overdraft
2. Kill the highest-interest credit card first — guaranteed 20%+ return
3. Maintain Robinhood habit ($454/mo) — keep this
4. THEN add new trading capital from genuine surplus

**The trading app helps most when you're trading from surplus, not from credit.**

---

## Summary

```
WHAT WE'RE BUILDING:
budget-app → exports your financial state
trading-app reads it → tells you what you can trade
you set rules → app tells you when conditions are met
you log trades → app shows you what's actually working

THE LOOP:
Know what you have → Set your rules → Act on signals → Track results → Improve rules
```
