# LongVol — Systematic Long Volatility Strategy

**Confidential — Gregory Rubio & Jakob — Do not distribute**

---

## Overview

LongVol is a systematic, signal-driven long volatility strategy trading ATM straddles on ETFs and single-name equities. The core thesis: volatility is periodically and measurably cheap relative to what will subsequently be realized. The strategy identifies these mispricings quantitatively, enters long straddle positions, manages delta exposure continuously, and exits when volatility has normalized or become rich.

**Universe:**
- **Phase 1:** ETFs (SPY, QQQ, IWM, GLD, TLT, XLF, XLE, and others from the 22-ETF surface dataset)
- **Phase 2:** Single-name equities (53 names with earnings data — earnings windows filtered out on entry)

**Structure:** Long ATM straddle (long call + long put, same strike, same expiry)
**Edge:** Buy realized vol that exceeds implied vol over the holding period, after accounting for theta bleed

---

## Strategy Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  1. CHEAP VOL IDENTIFICATION   → Is vol cheap enough to buy? │
│  2. MATURITY SELECTION         → Which expiry maximizes edge? │
│  3. ENTRY EXECUTION            → Enter ATM straddle           │
│  4. DELTA HEDGING              → EOD + intraday threshold      │
│  5. ONGOING VOL MONITORING     → Is vol still cheap? Stay in? │
│  6. EXIT EXECUTION             → Vol rich → close straddle     │
└─────────────────────────────────────────────────────────────┘
```

---

## 1. Cheap Volatility Identification

A position is only entered when a **composite cheapness score** crosses a threshold. No single metric is sufficient — the score aggregates multiple independent professional measures.

### 1.1 VRP Metrics (Implied vs Realized)

| Metric | Formula | Cheap Signal |
|--------|---------|--------------|
| **Raw VRP** | `IV_atm - RV_trailing` | VRP < threshold (IV < RV) |
| **VRP Z-score** | `(VRP - μ_VRP) / σ_VRP` rolling 252d | z < -1.0 |
| **RV/IV Ratio** | `RV_trailing / IV_atm` | Ratio > 1.10 (RV exceeds IV by 10%+) |
| **Conditional VRP** | VRP segmented by RV regime | Low-RV regime: IV collapse relative to typical |

**RV Calculation (no look-ahead):**
```
RV_n = sqrt(252 * sum(log_return_i^2, i=1..n) / n)
```
Windows: 5d, 10d, 21d, 63d — use the window that matches target holding period.

### 1.2 IV Percentile & Term Structure

| Metric | Definition | Cheap Signal |
|--------|-----------|--------------|
| **IV Percentile (IVP)** | Pct rank of current ATM IV vs trailing 252d | IVP < 25th pctile |
| **IV Rank (IVR)** | `(IV - IV_min) / (IV_max - IV_min)` trailing 252d | IVR < 20 |
| **Term Structure Slope** | `IV_60d / IV_30d` | Flat or inverted (≤ 1.0) = short end cheap |
| **Term Structure Z** | Z-score of slope vs 252d history | z < -1.0 |
| **Calendar Spread** | `IV_back - IV_front` | Positive but compressed = front cheap |

### 1.3 Vol-of-Vol & Skew

| Metric | Definition | Cheap Signal |
|--------|-----------|--------------|
| **VVIX/VIX ratio** (index) | Vol-of-vol relative to vol level | Low ratio = complacency, explosion risk |
| **Skew** | `IV_put_25d - IV_call_25d` | Low skew = cheap tail protection |
| **Skew Z-score** | Rolling 252d z of skew | z < -1.0 |
| **Butterfly (BF25)** | `(IV_25d_put + IV_25d_call)/2 - IV_50d` | Low BF = cheap wings = cheap straddle |
| **Smile Curvature** | 2nd derivative of IV smile | Flat smile = ATM relatively cheap |

### 1.4 Realized Vol Forecasting

| Metric | Definition | Cheap Signal |
|--------|-----------|--------------|
| **GARCH(1,1) forecast** | Forward RV estimate | `GARCH_RV > IV_atm` |
| **Parkinson RV** | High-low range estimator | Higher than close-to-close RV → IV underestimates |
| **Yang-Zhang RV** | Open/high/low/close estimator | Same |
| **Vol momentum** | Rate of change of RV_21d | Rising RV into cheap IV = strong buy |
| **Vol regime** | HMM state (low/med/high RV) | Transition from low→mid regime with IV lagging |

### 1.5 Microstructure & Flow

| Metric | Definition | Cheap Signal |
|--------|-----------|--------------|
| **Put/Call Vol Ratio** | Options volume put/call | Low = complacency, no tail hedging |
| **Gamma exposure (GEX)** | Dealer net gamma position | Large positive GEX = suppressed realized vol → reversal risk |
| **IV bid-ask spread** | `(IV_ask - IV_bid) / IV_mid` | Tight spread = liquid, low friction |

### 1.6 Composite Cheapness Score

Each metric is normalized to a z-score. The composite score is a weighted average:

```
Cheapness_Score = w1*VRP_z + w2*IVP_z + w3*TermStructure_z + w4*Skew_z + w5*BF25_z + w6*GARCH_z

Entry threshold: Cheapness_Score < -1.0  (at least 1 std cheap on composite)
Strong entry:    Cheapness_Score < -1.5
```

Initial weights: equal-weighted (1/6 each). Optimize via walk-forward on training data.

---

## 2. Maturity Selection

The optimal expiry balances:
1. **Edge per day** (VRP magnitude / DTE)
2. **Liquidity** (bid-ask spread as % of premium)
3. **Theta decay rate** (convexity of theta curve — avoid last 7 DTE)
4. **Gamma profile** (how much realized vol is captured per $ of premium)

### Rules

| Condition | Target Expiry |
|-----------|--------------|
| Normal contango term structure | 30-45 DTE — best theta/gamma ratio |
| Flat term structure | 20-30 DTE — faster vol realization |
| Inverted (backwardation) | 45-60 DTE — front too expensive, buy back month |
| Very cheap front-month (IVR < 10) | 14-21 DTE — maximize gamma per dollar |
| High VVIX (vol-of-vol spike) | 30 DTE — avoid front-month blow-up risk |

**Never enter with DTE < 10** — theta decay overwhelms any vol edge.

**Weekly vs monthly:** Use monthlies as primary. Weeklies only if front-month IVR < 10 AND expected holding period < 5 days based on signal strength.

### Expiry Score (rank all available expiries):
```
Expiry_Score = (VRP_edge / theta_per_day) * (1 / bid_ask_pct) * gamma_normalized
```
Select expiry with highest Expiry_Score within the allowed DTE range.

---

## 3. Entry Execution

### Strike Selection
- **ATM straddle:** Strike = closest listed strike to current underlying price
- Recheck at fill time — underlying may move between signal and execution
- Use mid-price limit orders. Never market orders on options.

### Sizing
```
Position_Size = Risk_Budget_per_trade / max_loss_scenario
max_loss_scenario = full premium paid (straddle cost = call_ask + put_ask)
Risk_Budget = f(portfolio_vol_target, current_drawdown)
```

### Entry Checklist
- [ ] Composite Cheapness Score < threshold
- [ ] DTE within target range
- [ ] Bid-ask spread < 2% of mid
- [ ] No earnings within 21 days (single names only — see Section 7)
- [ ] Not already in position for this ticker

---

## 4. Delta Hedging

The straddle starts delta-neutral at entry (ATM → delta ≈ 0). As the underlying moves, delta accumulates. We monetize gamma by re-hedging — this is the core P&L mechanism of the strategy.

### 4.1 EOD Hedge (always)

At market close every day the position is held:
```
Hedge_shares = -round(position_delta * contract_multiplier)
```
Execute via MOC (Market On Close) order for maximum liquidity and price efficiency. This locks in the day's gamma P&L and resets to delta-neutral overnight.

### 4.2 Intraday Threshold Hedge

Hedge intraday when the underlying has moved enough that the accumulated delta represents a meaningful profit worth locking in and the hedge cost is justified:

```
Intraday hedge trigger: cumulative move from last hedge ≥ ±1.5σ_5min
```

Where:
```
σ_5min = rolling 20-day standard deviation of 5-minute returns
Cumulative move = (current_price - last_hedge_price) / last_hedge_price
```

**Why 1.5σ?**
- At 1.0σ: over-hedging — transaction costs exceed gamma captured
- At 2.0σ: under-hedging — leaving delta risk on table, P&L variance too high
- 1.5σ: empirical sweet spot balancing gamma capture vs transaction drag (to be calibrated per ticker via backtest)

**Minimum size to hedge:** Do not execute a hedge for < 5 shares — transaction cost exceeds gamma captured on small moves.

### 4.3 Hedge Execution Logic

```python
# EOD: always hedge at close
# Intraday: only hedge if threshold crossed

current_delta = sum(option.delta * contracts * 100 for option in straddle)
cumulative_move_pct = (current_price - last_hedge_price) / last_hedge_price
move_in_sigma = cumulative_move_pct / sigma_5min

if abs(move_in_sigma) >= 1.5 or is_market_close:
    shares_to_trade = -round(current_delta)
    if abs(shares_to_trade) >= 5:
        execute_delta_hedge(shares_to_trade)
        last_hedge_price = current_price
        log_hedge_event(timestamp, shares_to_trade, gamma_pnl_captured)
```

### 4.4 Gamma Scalping P&L Attribution

Track separately per position:

| Component | Formula |
|-----------|---------|
| **Theta bleed** | `daily_theta * contracts * 100` (negative, daily cost) |
| **Gamma P&L** | Sum of `Δunderlying * shares_hedged` across all hedge events |
| **Vega P&L** | `ΔIV * vega * contracts * 100` |
| **Transaction costs** | Commissions + bid-ask slippage on hedges |
| **Net P&L** | Gamma P&L + Vega P&L − Theta bleed − Transaction costs |

If Gamma P&L consistently < Theta bleed over a rolling 10-day window → realized vol is not covering the cost → exit or reassess entry threshold.

---

## 5. Ongoing Vol Monitoring — Stay or Exit?

The position is re-evaluated at EOD (and intraday on large moves) against the same metrics used for entry, now framed as: **is vol still cheap enough to justify the remaining theta cost?**

### 5.1 Exit Signals (Vol Has Become Rich)

| Metric | Exit Trigger |
|--------|-------------|
| **VRP Z-score** | z > +0.5 (IV now above RV — vol has normalized) |
| **IV Percentile** | IVP > 75th pctile (vol now expensive) |
| **Composite Score** | Cheapness_Score > 0.0 (no longer cheap) |
| **Vega P&L** | Cumulative vega gain > 50% of original premium paid — take profits |
| **DTE** | DTE ≤ 7 — exit regardless (theta dominates, gamma collapses) |
| **Gamma/Theta ratio** | Rolling 5d Gamma_PnL / Theta_bleed < 0.5 — not covering cost |

### 5.2 Partial Exit Rules

- At 50% of max expected vega gain → close 50% of position, let rest run
- At DTE = 14 → evaluate: roll to next expiry vs closing completely
- If composite score has moved from strong cheap (< -1.5) to mild cheap (-1.0 to -0.5) → reduce size by 50%

### 5.3 Rolling Rules

Roll (close current + open new expiry) when:
- DTE ≤ 14 AND vol still cheap (Cheapness_Score < -0.5)
- New expiry has better Expiry_Score than current
- IV has NOT spiked intraday (avoid buying back expensive near-term straddle at wrong time)
- Roll cost (slippage + spread) < 3 days of expected gamma P&L

### 5.4 Stop Loss

- **Per position:** Max loss = 40% of premium paid → close position
- **Portfolio level:** 3 consecutive stop-losses → halve position sizing until win rate recovers over next 10 trades

---

## 6. P&L Decomposition (Required Reporting)

Every closed trade must record:

| Field | Description |
|-------|-------------|
| `ticker` | Underlying |
| `entry_date` / `exit_date` | Holding period |
| `expiry` | Option expiration |
| `dte_at_entry` | DTE when straddle opened |
| `entry_iv` | ATM IV at entry |
| `exit_iv` | ATM IV at exit |
| `realized_rv` | RV computed over holding period |
| `entry_cheapness_score` | Composite z-score at entry |
| `exit_cheapness_score` | Composite z-score at exit |
| `vega_pnl` | IV change × vega × contracts |
| `gamma_pnl` | Sum of all delta hedge P&L |
| `theta_bleed` | Cumulative theta paid |
| `transaction_costs` | Commissions + slippage |
| `net_pnl` | Total |
| `n_hedges` | Number of delta hedges executed |
| `exit_reason` | `vol_rich` / `dte_floor` / `stop_loss` / `vega_target` / `manual` |

---

## 7. Earnings Filter (Single Names Only)

Earnings events inflate IV structurally in the weeks before the announcement. Buying a straddle into earnings is an event-vol trade, not a structural cheap-vol trade — it has fundamentally different risk/reward and must be excluded from this strategy.

### Rules

1. **No new entries within 21 days before earnings** — event premium inflates IV, cheapness score becomes unreliable
2. **If already in position:** exit by 14 days before earnings — avoid holding through the event
3. **Post-earnings re-entry:** wait 5 calendar days after earnings before re-evaluating — IV crush takes 2-3 days to stabilize, data needs to recalibrate
4. **Earnings calendar:** wire from `Earnings_Trader_Options/` data (53 names available) for backtesting; use real-time calendar API for live

```python
# Entry gate — single names only
days_to_earnings = next_earnings_date(ticker) - today
if days_to_earnings < 21:
    return SKIP  # Do not enter — event premium contaminates cheapness score
```

**Note: This filter does NOT apply to ETFs** — ETF IV is driven by basket dynamics, not single earnings events.

---

## 8. Data Sources

| Data | Source | Coverage | Size |
|------|--------|----------|------|
| ETF 5-min option surfaces (22 ETFs) | HDD: `/Volumes/EXTERNAL_US/Cross_asset_Surfaces_5min/` | Jan 2020 – Dec 2025 | ~125 GB (parquet) |
| Single-name earnings surfaces (53 names) | OneDrive: `Earnings_Trader_Options/data_theta_v3/` | Q1 2022 – Q2 2025 | 64 GB |
| SPY/SPXW 1-min underlying | OneDrive: `Trading_Platform/DATA/` | 2022 – present | ~12 GB |
| VIX daily surfaces | OneDrive: `ShortTheVix/DATA/VIX_Options/` | 2017 – 2025 | — |
| VIX futures 1-min | OneDrive: `ShortTheVix/DATA/` | 204 contracts | — |

All ETF surface files are in **parquet format** (converted March 2026).

Available columns per row: `trade_date, expiration, dte, symbol, strike, right, timestamp, bid, ask, delta, theta, vega, rho, gamma, vanna, charm, vomma, implied_vol, underlying_price` + higher-order Greeks.

---

## 9. Research Pipeline

```
Phase 1 — ETF universe
  [ ] Build composite cheapness score from ETF surfaces
  [ ] Backtest entry/exit signals (in-sample: 2020-2022)
  [ ] Calibrate delta hedge threshold (1.5σ starting point, per-ticker optimization)
  [ ] Validate maturity selection rules
  [ ] OOS test: 2023-2024
  [ ] Walk-forward validation + multiple-testing correction (DSR)

Phase 2 — Single names
  [ ] Wire earnings calendar
  [ ] Apply 21-day earnings filter
  [ ] Extend cheapness score to single names (idiosyncratic RV)
  [ ] Backtest single names OOS

Phase 3 — Portfolio construction
  [ ] Correlation analysis across positions (avoid crowded long-vol book)
  [ ] Position sizing optimizer (vol-targeting)
  [ ] Portfolio-level delta hedging (net delta across all positions)
  [ ] Live paper trading (5+ sessions)
  [ ] Live deployment
```

---

## 10. Key Risks & Mitigations

| Risk | Description | Mitigation |
|------|-------------|-----------|
| **Theta bleed with no RV** | IV cheap but RV never materializes | Strict Cheapness_Score threshold + 40% stop loss |
| **IV crush after entry** | IV drops further — vega loss > gamma gain | Daily composite score monitoring; exit if no longer cheap |
| **Earnings surprise** | Single-name event vol contaminates signal | 21-day pre-earnings entry block |
| **Liquidity / wide spreads** | Slippage eats edge | Bid-ask < 2% of mid filter at entry |
| **Over-hedging** | Transaction costs > gamma captured | Min 1.5σ move before intraday hedge; min 5-share threshold |
| **Under-hedging** | Delta accumulates, straddle becomes directional | Hard EOD hedge every day |
| **Crowded long-vol book** | Positions all move together in sell-off | Max 3 correlated positions; correlation check at entry |
| **Vol regime change** | Low-vol regime extends indefinitely | Rolling composite score check; exit when score > 0 |

---

*Confidential — Gregory Rubio & Jakob — Internal use only*
