# trading-app

Budget-aware trading advisor. Single HTML file, no backend.

## How to Use
Open `index.html` in a browser. All data saved to localStorage.

## Key Concepts
- **Capital Gate** — shows how much you can trade based on Robinhood balance + risk profile
- **Rules Engine** — you define the conditions that trigger a trade (no impulse trading)
- **Signal Check** — enter today's market conditions, see which rules fire
- **Trade Journal** — log every trade, track real P&L over time
- **Performance** — win rate, avg P&L, rule-level breakdown

## Budget App Integration
1. Open `budget-app/index.html` in the same browser
2. Click "Export for Trading" (see TRADE_PLAN.md Phase 2 for when this is built)
3. Trading app auto-reads `romeo_financial_state` from localStorage
4. Capital Gate syncs to your real surplus

## Full Plan
See `TRADE_PLAN.md` — complete integration roadmap, all 5 phases.

## Warning
Trade only from genuine surplus. Pay debt first. No emergency fund = no aggressive trades.
