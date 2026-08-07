# Fundamental & Sectoral Integration Architecture

This document outlines the proposed architectural design for integrating the `screener_stocks` CSV data into the Apollo Backtest Engine. Currently, Apollo is a highly tuned 100% technical momentum engine. By integrating this data, Apollo evolves into a **Techno-Fundamental** system, capable of confirming technical breakouts with underlying business realities and sectoral tailwinds.

---

## 1. The Data Inputs

The daily reports provide us with a goldmine of data for each stock. We will focus on extracting and utilizing the following core metrics:

### Fundamental Health Metrics
- **Growth Engine:** `Profit Growth YoY (%)` & `Revenue Growth YoY (%)`
- **Quality & Efficiency:** `Return on Equity`
- **Risk / Balance Sheet:** `Debt to Equity` & `Free Cash Flow`
- **Valuation:** `P/E Ratio`

### Macro / Contextual Metrics
- **Theme Tracking:** `Sector`
- **Size Profiling:** `Market Cap (Cr)`

---

## 2. Core Architectural Components

To process this data efficiently without slowing down Apollo's rapid technical scanning, we will build three new architectural components:

### A. The Fundamental Data Repository (In-Memory)
Loading CSVs during the backtest loop would cripple the engine's speed.
- **Design:** We will build a new module (e.g., `apollo_core/fundamental_repo.py`) that loads the latest `screener_stocks*.csv` files *once* when the engine boots.
- **Mechanism:** It will parse the CSVs, handle missing values (NaNs), and build a fast O(1) dictionary mapping: `Symbol -> Fundamental Dict`.

### B. The Fundamental Quality Score (FQS)
Instead of looking at raw numbers (which are hard to process systematically), we will pass the metrics through a scoring matrix to generate a **Fundamental Quality Score (0 to 100)** for every stock.

**Proposed Scoring Logic (Example):**
- **Growth (40 pts):** `Profit Growth YoY > 20%` (+20 pts), `Revenue Growth YoY > 15%` (+20 pts).
- **Quality (30 pts):** `ROE > 15%` (+15 pts), Positive `Free Cash Flow` (+15 pts).
- **Risk Penalty (Negative pts):** `Debt to Equity > 2.0` (Subtract 30 pts or hard block).

### C. The Sectoral Heatmap Manager
Technicals tell us *when* a stock is moving; sectors tell us *why*. If multiple stocks in the same sector break out simultaneously, it's institutional rotation.
- **Design:** The engine will group the daily technical scores by the `Sector` column.
- **Mechanism:** We calculate the percentage of stocks within a sector that are currently in the "Trending Up (PRIME or CONFIRMED)" buckets. If a sector has high participation, it is marked as a **"Hot Sector"**.

---

## 3. Integration into Apollo's Decision Making

How will this actually affect the trades Apollo takes? We will integrate this data at **Layer 3: The Entry Gate** (inside `extract_trades()` in `trade_engine.py`).

### I. The Fundamental Conviction Modifier
When Apollo detects a technical entry signal (Score > Entry Threshold), it will query the Fundamental Quality Score (FQS) and apply a modifier:
1. **The "Conviction Boost":** If FQS > 75 (Stellar Fundamentals), we lower the technical entry threshold by 5 points. The engine will aggressively front-run entries on phenomenal companies.
2. **The "Toxic Block":** If FQS < 25 (e.g., Negative Profit Growth, High Debt), we raise the technical entry threshold to 90 or block the trade entirely. We refuse to buy technical breakouts on fundamentally dying companies, saving us from false breakouts/traps.

### II. The Sectoral Tailwind Bonus
If a technical entry triggers on a stock that belongs to a **"Hot Sector"** (e.g., Defense or Railways during a theme run), the trade is immediately flagged as a `THEMATIC BREAKOUT`.
- **Action:** The system will attach a `+10` bonus to the final conviction score, prioritizing these trades over isolated breakouts in cold sectors.

---

## 4. UI & Output Enhancements
The `extract_trades()` output dictionaries will be expanded. When you view the backtest results or active signals, you will now see:
- `Fundamental_Score` (e.g., 85/100)
- `Sector` (e.g., "Capital Goods")
- `Is_Sector_Hot` (True/False)
- `Fundamental_Flags` (e.g., "WARNING: High Debt", "STAR: 30% YoY Profit")

This ensures that the final user (you) has the complete Techno-Fundamental picture before authorizing a live trade.

---

> [!IMPORTANT]
> **User Review Required:** 
> Please review this architecture. How would you like to tune the Fundamental Quality Score? Are there specific metric thresholds (e.g., ROE must be strictly > X%) that you want to act as absolute hard blocks, regardless of how good the technicals look? What additions or modifications would make this system most efficient for your specific decision-making process?

GLM'S Response:

Now I have a complete picture of everything in that zip — all 20 CSVs, their schemas, and the architecture doc. Let me give you a structured, critical breakdown.
What You Actually Have: Data Inventory

Your daily reports contain four distinct signal families across 20 CSVs:
Signal Family
	
Files
	
What It Gives You
	
Stocks/File
Fundamental Snapshots	6 × screener_stocks	P/E, ROE, D/E, FCF, Profit/Revenue Growth YoY, Sector, Market Cap, 52W High/Low	34–100
Multi-TF RSI	6 × RSI Dashboard (15M, 30M, 1HR, 4HR, Daily, Weekly)	RSI 21/36/56 across 6 timeframes — exact same periods Apollo uses	500–1000
Multi-TF Momentum	6 × Nifty 500 Gainers (1D, 2D, 3D, 1W, 2W, Monthly)	% return + close price at each lookback	50–250
Comprehensive Momentum Profile	1 × Nifty Cash Gainers	8 timeframes of % returns (2D through 56D) + closing prices at each lookback	500
  
Critical Opinion on the Architecture Doc

The fundamental_integration_architecture.md proposes an FQS (Fundamental Quality Score) and a Sectoral Heatmap. The direction is right. The execution has issues.
What It Gets Right

Sectoral Heatmap concept — If 7 out of 10 Capital Goods stocks are trending up simultaneously, that's institutional sector rotation, not individual stock noise. This is the highest-value idea in the doc. No amount of technical analysis on a single stock captures this.

In-memory repository design — Loading CSVs once at boot instead of inside the backtest loop is the correct engineering choice. Obvious but necessary.
What It Gets Wrong

1. The FQS is Hand-Tuned Guesswork Dressed Up as a Score

The doc proposes: Profit Growth >20% → +20 pts, Revenue Growth >15% → +20 pts, ROE >15% → +15 pts, etc. These thresholds are pulled from thin air. There's no justification for why 20% profit growth is "good" vs 19%, or why it gets exactly 20 points vs 15.

But the deeper problem is more fundamental: this is a single-day snapshot, not a time series. You have one screener_stocks file per day. You can't compute "trend in ROE" or "improving profit growth" because you have exactly one data point per stock per metric. You're scoring a company's entire fundamental health based on a single day's reported numbers, which are themselves:

     Trailing (last 4 quarters, not forward-looking)
     Stale (reported quarterly, not real-time)
     Already priced in (fundamental data is public information — the market knows ROE is 25% before you do)

My take: FQS as a hard gate ("block trade if FQS < 25") is dangerous because single-snapshot fundamentals can't reliably separate winners from losers. As a soft modifier (+/- a few points to conviction), it's harmless but also possibly valueless. The burden of proof is on showing it improves OOS returns.

2. The "Conviction Boost" (Lower Threshold for High FQS) Is an Unvalidated Assumption That Could Hurt

The doc says: if FQS > 75, lower the technical entry threshold by 5 points. This means you'd enter trades on fundamentally strong companies with weaker technical signals. The implicit assumption is "good fundamentals + mediocre technicals = better trade than mediocre fundamentals + strong technicals." There's zero evidence presented for this, and in momentum-driven markets (which NSE frequently is), it's often backwards. Strong momentum stocks frequently have terrible fundamentals (high P/E, low ROE, high debt) because the market is pricing in future earnings that haven't materialized yet. Think of small-cap infrastructure or defense stocks in 2023-24 — many had negative FCF and high P/E but were the best momentum trades of the year.

3. The "Toxic Block" (FQS < 25 → Block Trade) Will Cause Missed Trades

A hard block on fundamentally weak stocks would have kept you out of some of NSE's biggest momentum moves. SAPPHIRE FOODS in your own data: P/E = 0 (loss-making), ROE = 0%, negative Profit Growth — by any fundamental scoring this would be "toxic." It gained 12.26% in a single day and 25.18% in a week. A momentum engine that blocks this trade has failed at its primary job.
What the Architecture Doc Completely Misses

The doc focuses exclusively on screener_stocks (fundamentals) and ignores the three richer data sources in your daily reports. Here's where the real value lies:
1. Intraday RSI Progression Profile (THE highest-value data you're not using)

You have RSI 21/36/56 at six timeframes on the same day: 15M → 30M → 1HR → 4HR → Daily → Weekly. This is a time-resolved RSI trajectory for every stock, and it enables signals that neither Apollo's EOD data nor any single-timeframe analysis can produce:

RSI Intraday Divergence: If a stock's 15M RSI21 is 99 (parabolic) but its Daily RSI21 is only 65, the surge is intraday-only. If you're an EOD scanner, this stock might not even appear on your radar yet — but the intraday RSI tells you something is building. Conversely, if 15M RSI21 = 60 but Daily RSI21 = 88, the daily trend is extended but intraday momentum is fading — a classic topping pattern.

RSI Timeframe Convergence Score: For each stock, compute how tightly the RSI values cluster across timeframes:
python
 
  
 
 
# Lower = more consistent trend across timeframes
rsi_spread = std([RSI21_15m, RSI21_30m, RSI21_1h, RSI21_4h, RSI21_daily, RSI21_weekly])
 
 

A stock with RSI21 = [88, 85, 83, 81, 80, 78] across timeframes has a tight, consistent uptrend. A stock with RSI21 = [99, 72, 55, 48, 44, 41] has a speculative intraday spike on top of a weak broader trend. This is a genuinely independent signal that doesn't exist in Apollo's current EOD-only pipeline.

Apollo Integration: This doesn't go into the backtest (it's intraday). It goes into your live signal generation — as a pre-filter or confidence modifier for the next day's EOD scan.
2. Multi-Timeframe Momentum Profile from Nifty Cash Gainers

This single file is the most underappreciated asset in your reports. It has % return and closing price across 8 lookbacks (2D, 3D, 5D, 10D, 21D, 36D, 56D) for 500 stocks. This enables:

Momentum Acceleration/Deceleration: Not just "is it going up?" but "is the rate of change increasing?"
python
 
  
 
 
# Simple example: 5D return > 10D return > 21D return = accelerating momentum
acceleration = (ret_2d > ret_3d > ret_5d > ret_10d > ret_21d)
 
 

Stocks where short-term returns exceed long-term returns are in an accelerating uptrend — the strongest momentum regime. Stocks where 2D return is positive but 5D/10D are flat are one-day spikes.

Momentum Regime Classification: Using the 8-timeframe return vector, you can classify each stock into:

     Accelerating (2D > 5D > 21D return)
     Sustained (all timeframes positive, stable)
     Decelerating (21D return high, but 2D return fading)
     Dead Cat Bounce (2D/5D positive, but 36D/56D deeply negative)

This is far more granular than Apollo's binary TRENDING_UP / SIDEWAYS / DOWNTREND classification and can feed directly into the tier system we discussed.

Proximity to 52-Week High: From screener_stocks, you have 52 Week High and Last Price. Distance to 52W high is one of the most robust factors in academic momentum literature (George and Hwang, 2004; 52-week high effect). Stocks within 5-10% of their 52W high statistically outperform. This is a trivially computable signal from data you already have.
3. Cross-Report Stock Intersection (The Multi-Signal Confluence)

The most powerful use of these reports isn't any single file — it's the intersection. Build a Venn diagram approach:

     Set A: Stocks in Top 10 of Nifty Cash Gainers (strong multi-TF momentum)
     Set B: Stocks with RSI Stack Intact across Daily + Weekly (RSI21 > RSI36 > RSI56 > 50 on both timeframes)
     Set C: Stocks with FQS > 60 (reasonable fundamentals)
     Set D: Stocks in a "Hot Sector" (sector heatmap > threshold)

A ∩ B = Technically strongest momentum stocks with confirmed trend across timeframes. This is your highest-conviction technical set.

A ∩ B ∩ C = Technically strong + fundamentally sound. Small set, highest conviction for larger position sizes.

A ∩ B ∩ D = Technically strong + sector tailwind. Highest conviction for thematic trades.

A ∩ B ∩ C ∩ D = The rarest set — maybe 2-5 stocks per day. But these are the ones worth aggressive allocation.

The beauty of this approach is that each set is independently useful, and the intersection provides natural conviction scaling without needing arbitrary score thresholds.
Concrete Integration Proposal for Apollo

Here's how I'd actually integrate this, prioritized by value-to-effort ratio:
Phase 1: Cross-Validation Layer (Low Effort, High Value)

     Compare Chartink RSI values with Apollo's computed RSI for the same stock/date. Discrepancies >5 points indicate data quality issues in either source. This is a data integrity check, not a trading signal.
     Use Nifty Cash Gainers' 8-timeframe returns as a pre-computed momentum profile instead of Apollo computing 8 separate rolling returns. Saves computation and provides an external validation point.

Phase 2: Sector Rotation Engine (Medium Effort, High Value)

     From screener_stocks: Build a daily sector-level aggregation. For each sector, compute: % of stocks positive, average % change, median RSI, number of new 52W highs.
     Track sector momentum over time (requires storing daily sector aggregates). Sectors moving from cold → warm → hot are early-entry opportunities.
     Feed sector score into trade conviction as a multiplier, not a gate. Hot sector = 1.1x conviction. Cold sector = 0.9x. This is gentler than the doc's "THEMATIC BREAKOUT +10" approach and won't cause you to miss isolated breakouts in cold sectors.

Phase 3: Intraday RSI Profiler (Medium Effort, Medium-High Value — Live Only)

     Parse the 6 RSI Dashboard files daily. Build an RSI progression object per stock: {timeframe: {rsi21, rsi36, rsi56}}.
     Compute the RSI Timeframe Convergence Score (standard deviation across timeframes). Low convergence = consistent trend = higher confidence.
     Detect intraday divergence: 15M RSI21 > 90 but Daily RSI21 < 60 → flag as "speculative intraday spike, not yet confirmed on daily."
     Use as next-day pre-filter: If a stock has strong intraday RSI convergence AND appears in Apollo's EOD scan, elevate its priority.

Phase 4: Fundamental Overlay (Higher Effort, Lower Certainty of Value)

     Implement FQS but ONLY as a soft modifier (±3-5 points to conviction score), never as a hard gate.
     Add 52W High proximity (distance % from 52W high) as a separate, well-studied factor. This is more robust than P/E or ROE for momentum systems.
     Use Market Cap from screener_stocks for size-based position sizing (smaller caps = smaller positions). This is practical risk management, not alpha generation.

What I'd Explicitly NOT Do

     Don't hard-block any trade based on fundamentals. The doc's "Toxic Block" (FQS < 25 → block) will cause missed trades in momentum regimes where the best performers have the worst fundamentals.
     Don't lower technical thresholds for high-FQS stocks. The "Conviction Boost" assumes fundamentals predict technical success, which is backwards for momentum systems.
     Don't build the FQS with arbitrary thresholds. If you build it, derive the thresholds from data (e.g., median ROE of NIFTY 500 stocks = your baseline, not 15%).

Bottom Line

The architecture doc sees only one-third of the value in your daily reports. The screener_stocks fundamentals are the weakest signal source you have. The strongest signals are: the multi-timeframe RSI progression (intraday trajectory), the 8-timeframe momentum profile (Nifty Cash Gainers), and the cross-report intersection logic (multi-signal confluence). I'd prioritize building the sector rotation engine and the RSI convergence profiler before touching the FQS at all. And if you only build one thing today, build the cross-report intersection — it turns six independent data dumps into a single prioritized watchlist with built-in conviction scaling.