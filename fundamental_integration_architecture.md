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
