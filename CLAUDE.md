# trading-app

Personal trading advisor — budget-aware capital management, news rules engine, trade journal.

## Purpose
Guide smart trading decisions based on real financial position from budget-app.
See `TRADE_PLAN.md` for full architecture and integration spec.

## Stack
- Level 1: Single HTML file (`index.html`)
- No backend, no deps, all localStorage
- Reads `romeo_financial_state` from localStorage (written by budget-app)
- Upgrade path: Level 3 with Flask backend + news feeds + Claude API

## Data Schema
All state stored in localStorage:
- `romeo_financial_state` — synced from budget-app (capital, risk flags)
- `trading_rules` — user-defined trade trigger rules (array)
- `trading_positions` — open positions
- `trading_journal` — completed trades
- `trading_profile` — risk profile + preferences
- `trading_watchlist` — tracked tickers

## Modules
1. Capital Gate — how much can I trade?
2. Rules Engine — what conditions trigger action?
3. Trade Advisor — specific signals + position sizer
4. Trade Journal — logged trades + performance

## Budget App Integration
Budget app writes `romeo_financial_state` to localStorage.
Trading app reads it on load, updates capital display.
See `TRADE_PLAN.md` Phase 3 for full bridge spec.

## Design
Matches budget-app design system: dark warm theme, amber accent, Bricolage + Space Grotesk fonts.
