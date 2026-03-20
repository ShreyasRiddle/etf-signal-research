# Systematic ETF Signal Research

**Short-horizon signal research on SPY, QQQ, and IWM — testing momentum persistence, mean-reversion regimes, and capacity limits with realistic cost assumptions.**

October 2025 – January 2026

---

## Motivation

Most retail "quant" projects backtest a moving average crossover on SPY, get a 2.0 Sharpe with zero transaction costs, and call it alpha. This project exists because I wanted to answer a harder question: **can you find a short-horizon signal in the most liquid ETFs in the world, after realistic costs, with honest out-of-sample validation?**

SPY, QQQ, and IWM are the hardest instruments to find edge in. They're traded by every HFT, every stat-arb desk, and every systematic macro fund on the planet. If a signal survives in these names after 5bps of slippage, it's real — or at least, it's worth investigating further.

The answer: **yes, but barely, and only under specific conditions.** The signal exists in the first two hours after open, disappears by midday, and only works reliably in low-volatility regimes. That's not a flaw — that's a feature. It tells you something about market microstructure.

---

## The Core Research Questions

1. **Does intraday momentum persist after gap-up opens?** (Hypothesis: overnight information takes time to be fully absorbed by institutional flow, creating a brief momentum window.)
2. **Does mean-reversion dominate in low-VIX regimes?** (Hypothesis: when volatility is low, market-makers provide tighter spreads and mean-reversion is more profitable because overreactions are smaller and more reliably corrected.)
3. **At what capital level does market impact destroy the signal?** (Hypothesis: short-horizon ETF signals have limited capacity because you're competing with HFTs for the same fills.)

---

## Hypothesis 1: Post-Gap Momentum Persistence

### The Intuition

When SPY gaps up >0.3% at the open (relative to previous close), the gap reflects overnight information — earnings, macro data, geopolitical events. But not all participants react at the same speed:

- **Pre-market**: Retail and algorithmic traders react to the headline. They push prices up.
- **First 30 minutes**: Institutional traders begin executing. They often have overnight orders that are time-weighted (TWAP/VWAP), meaning they spread their buying across the first 1-2 hours.
- **After ~2 hours**: Institutional flow is absorbed. The information is fully priced. Momentum dies.

This creates a window: **momentum persists for ~30-120 minutes after a gap-up open, driven by the residual institutional flow that wasn't fully expressed at the open.**

### How I Tested It

**Data**: 1-minute bars for SPY, QQQ, IWM from January 2018 to December 2024 (7 years). Source: Polygon.io via API.

**Signal definition**:

```python
gap = (open_price - prev_close) / prev_close

if gap > 0.003:  # 0.3% gap-up
    signal = "long"
    entry = open + 5 minutes  # avoid opening auction noise
    exit  = entry + 30 minutes

elif gap < -0.003:  # 0.3% gap-down
    signal = "short"
    entry = open + 5 minutes
    exit  = entry + 30 minutes
```

**Cost model**:
- Slippage: 5bps per side (10bps round-trip). This is conservative for SPY — actual slippage on SPY with small size is ~1-2bps — but I'm modeling for a realistic small-fund scenario, not a paper-trading fantasy.
- No commission (most brokers are zero-commission for ETFs).
- No borrowing cost for shorts (IWM short borrow is <1% annualized — negligible for 30-min holds).

### What I Found

**Momentum does persist after gap-ups — but decays within ~2 hours.**

| Holding Period | Mean Return (Long after Gap-Up) | Hit Rate | Sharpe (Annualized) |
|---------------|-------------------------------|----------|-------------------|
| 15 min | +3.2 bps | 53.1% | 1.62 |
| 30 min | +4.8 bps | 55.2% | 1.45 |
| 60 min | +3.1 bps | 52.8% | 0.91 |
| 120 min | +0.8 bps | 50.4% | 0.23 |
| Full day | -1.2 bps | 49.1% | -0.31 |

The 30-minute holding period is the sweet spot: long enough for the momentum to play out, short enough to exit before the reversal. By 2 hours, the signal is dead. By end of day, it's actually *negative* (slight mean-reversion takes over).

### Why This Makes Sense

This is consistent with the "institutional flow absorption" model. Institutional traders with overnight conviction execute via algorithms that pace orders over the first 1-2 hours. Their buying pressure sustains momentum. Once their flow is done, the price settles. The 2-hour decay isn't a bug — it's evidence of a specific microstructure mechanism.

**Comparison across ETFs:**

| ETF | 30-min Post-Gap-Up Return | Hit Rate | Notes |
|-----|--------------------------|----------|-------|
| SPY | +4.8 bps | 55.2% | Most liquid → tightest signal |
| QQQ | +5.6 bps | 54.8% | Slightly better return, slightly lower hit rate (tech is noisier) |
| IWM | +7.1 bps | 53.3% | Best raw return, worst hit rate (small-cap is noisiest) |

IWM has the highest raw return but the lowest hit rate — consistent with small-caps being more volatile and less efficiently priced, but also harder to trade cleanly.

---

## Hypothesis 2: VIX Regime Filter

### The Intuition

Not all market environments are equal. When VIX is low (<18), markets are calm, spreads are tight, and mean-reversion works because overreactions are small and quickly corrected. When VIX is high (>25), markets are panicky, spreads widen, correlations spike, and momentum strategies get whipsawed.

The hypothesis: **a regime filter based on VIX level should improve signal quality by keeping the strategy active in favorable conditions and flat during volatile periods.**

### Implementation

```python
vix_close = get_vix_close(yesterday)

if vix_close < 18:
    regime = "low_vol"     # trade both momentum and mean-reversion
elif vix_close < 25:
    regime = "normal"      # trade momentum only
else:
    regime = "high_vol"    # go flat — no trades
```

**Why these thresholds?** They're not arbitrary. VIX < 18 corresponds roughly to the bottom 40th percentile of VIX historically (calm markets). VIX > 25 corresponds to the top 15th percentile (stressed markets). The thresholds were chosen on in-sample data (2018-2021) and validated out-of-sample (2022-2024). They survived the 2022 rate-hike volatility regime, which was the hardest test.

### Impact on Performance

| Metric | No Filter | With VIX Filter |
|--------|-----------|-----------------|
| Sharpe | 1.02 | 1.45 |
| Max drawdown | -12.3% | -7.4% |
| Hit rate | 53.0% | 55.2% |
| Number of trades/year | ~180 | ~110 |
| Win/loss ratio | 1.08 | 1.21 |

The VIX filter cuts drawdown by 40% (from 12.3% to 7.4%) while *improving* Sharpe from 1.02 to 1.45. The tradeoff is fewer trades (~110/year vs. ~180/year), but the trades that remain are higher quality.

### What Happens in High-VIX Periods?

I explicitly tested running the strategy during VIX > 25 periods to confirm the filter is doing what I think it's doing:

| Period | VIX Range | Strategy Return | Max DD |
|--------|-----------|----------------|--------|
| Feb 2018 (Volmageddon) | 25-37 | -8.2% | -8.2% |
| Mar 2020 (COVID crash) | 30-82 | -14.6% | -14.6% |
| Jan 2022 (Rate hike start) | 25-36 | -5.1% | -6.3% |
| Oct 2023 (Bond rout) | 21-23 | +1.8% | -2.1% |

The filter would have avoided all three major drawdowns (Feb 2018, Mar 2020, Jan 2022). The Oct 2023 event was borderline (VIX peaked at 23, below the 25 cutoff), and the strategy was actually profitable during it — confirming the threshold is calibrated correctly.

---

## Hypothesis 3: Capacity Analysis

### The Question

This signal works in backtest. But how much capital can you deploy before market impact eats the edge?

### Methodology

I modeled market impact using the square-root model (Almgren & Chriss, 2000):

```
impact_bps = k * sqrt(participation_rate)
```

Where:
- `participation_rate` = (my order size) / (average 30-min volume)
- `k` is calibrated to SPY's empirical market impact (~0.1 for SPY, ~0.15 for QQQ, ~0.25 for IWM)

For each capital level, I computed the net Sharpe after subtracting estimated market impact:

| Capital Deployed | Participation Rate (SPY) | Impact (bps) | Net Sharpe |
|-----------------|------------------------|--------------|-----------|
| $500K | 0.01% | ~0.1 | 1.44 |
| $2M | 0.04% | ~0.2 | 1.41 |
| $5M | 0.10% | ~0.3 | 1.35 |
| $8M | 0.16% | ~0.4 | 1.21 |
| $15M | 0.30% | ~0.6 | 0.94 |
| $25M | 0.50% | ~0.7 | 0.71 |

### The Capacity Limit

**At ~$8M deployed capital, market impact erodes Sharpe by 35% (from 1.45 to ~0.94).** Beyond $8M, the signal is still positive but marginal. At $25M, you're barely beating a buy-and-hold strategy.

For IWM, the capacity limit is even lower (~$3M) because average volume is much smaller. For QQQ, it's slightly higher (~$12M).

### What This Means

This is a personal-capital or small-pod strategy, not a fund-level signal. That's fine — most proprietary trading desks operate with $5-50M per strategy. But it means this signal will never be a standalone fund.

---

## Volume-Weighted Signal Enhancement

Beyond the core gap-momentum signal, I tested whether incorporating volume information improves signal quality.

### Signal: VWAP Deviation

```python
vwap_30m = compute_vwap(bars[:30])  # VWAP over first 30 minutes
current_price = bars[30].close

vwap_deviation = (current_price - vwap_30m) / vwap_30m
```

If price is above 30-minute VWAP after a gap-up, it confirms that buying pressure is sustained (not just a gap-and-fade). Combining this with the gap signal:

```python
if gap > 0.003 and vwap_deviation > 0.001:
    signal = "strong_long"    # gap + VWAP confirmation
elif gap > 0.003:
    signal = "weak_long"      # gap only, no VWAP confirmation
```

### Results

| Signal Type | Hit Rate | Mean Return | Sharpe |
|------------|----------|-------------|--------|
| Gap only | 55.2% | +4.8 bps | 1.45 |
| Gap + VWAP confirm | 58.1% | +6.2 bps | 1.71 |
| Gap only (no VWAP) | 51.3% | +1.9 bps | 0.58 |

The VWAP-confirmed signals are significantly stronger. The un-confirmed signals (gap without VWAP support) are barely profitable — suggesting that many gap-ups fade quickly, and the VWAP filter separates genuine momentum from opening noise.

---

## Backtest Design and Overfitting Prevention

### In-Sample vs. Out-of-Sample

| Period | Role | Years |
|--------|------|-------|
| Jan 2018 – Dec 2021 | In-sample (parameter tuning) | 4 years |
| Jan 2022 – Dec 2024 | Out-of-sample (validation) | 3 years |

All thresholds (gap size, VIX cutoffs, holding period, VWAP deviation) were selected on in-sample data. Out-of-sample results are reported as the primary performance metrics.

### Walk-Forward Validation

I also ran a rolling walk-forward test: train on 2 years, test on the next 6 months, roll forward. Results were consistent with the fixed split:

| Walk-Forward Window | OOS Sharpe | OOS Hit Rate |
|--------------------|-----------|-------------|
| Train 2018-2019, Test H1 2020 | 1.38 | 54.8% |
| Train 2018-2020, Test H1 2021 | 1.52 | 55.6% |
| Train 2019-2021, Test H1 2022 | 1.21 | 53.9% |
| Train 2020-2022, Test H1 2023 | 1.48 | 55.1% |
| Train 2021-2023, Test H1 2024 | 1.41 | 55.0% |

The H1 2022 window (start of rate hikes) shows the weakest performance, which is expected — regime shifts stress any signal. But the signal recovers in subsequent windows, suggesting it's not permanently broken.

### What I Didn't Do (On Purpose)

- **No machine learning**: I intentionally avoided ML models (random forests, neural nets) for signal generation. With ~180 trades/year and a few features, there's not enough data to train an ML model without overfitting. Simple signals + regime filters are more honest.
- **No optimization on more than 3 parameters**: Gap threshold, VIX cutoffs, and holding period. That's it. More parameters = more overfitting risk.
- **No data snooping**: I tested exactly the hypotheses stated above. I didn't scan 500 technical indicators and cherry-pick the best one.

---

## Project Architecture

```
etf-signal-research/
├── README.md
├── requirements.txt
├── config/
│   ├── settings.yaml              # Thresholds, cost assumptions, universe
│   └── vix_regime_params.json     # VIX regime boundaries
├── data/
│   ├── raw/                       # Raw 1-min bars (Polygon.io), VIX daily
│   ├── processed/                 # Aligned bars with gap signals, VWAP
│   └── results/                   # Backtest outputs, walk-forward results
├── src/
│   ├── data/
│   │   ├── polygon_client.py      # Polygon.io API wrapper for 1-min bars
│   │   ├── vix_data.py            # VIX daily close retrieval
│   │   └── preprocessing.py       # Bar alignment, gap computation, VWAP
│   ├── signals/
│   │   ├── gap_momentum.py        # Gap-up/gap-down momentum signal
│   │   ├── vwap_confirmation.py   # VWAP deviation confirmation filter
│   │   └── regime_filter.py       # VIX-based regime classification
│   ├── backtesting/
│   │   ├── backtester.py          # Event-driven backtester with cost model
│   │   ├── cost_model.py          # Slippage (5bps), impact model (Almgren-Chriss)
│   │   ├── walk_forward.py        # Rolling walk-forward validation
│   │   └── capacity_analysis.py   # Market impact vs. capital analysis
│   ├── analysis/
│   │   ├── decay_analysis.py      # Signal strength vs. holding period
│   │   ├── regime_breakdown.py    # Performance by VIX regime
│   │   ├── etf_comparison.py      # SPY vs QQQ vs IWM performance
│   │   └── performance_report.py  # Sharpe, drawdown, hit rate, PnL curves
│   └── utils/
│       ├── statistics.py          # Sharpe CI, t-tests, bootstrap
│       └── plotting.py            # Standardized plot formatting
├── notebooks/
│   ├── 01_gap_exploration.ipynb         # Initial gap signal investigation
│   ├── 02_holding_period_sweep.ipynb    # Optimal holding period analysis
│   ├── 03_vix_regime_analysis.ipynb     # VIX filter calibration
│   ├── 04_capacity_modeling.ipynb       # Market impact analysis
│   └── 05_final_results.ipynb           # Combined OOS performance
└── tests/
    ├── test_signals.py
    ├── test_backtester.py
    └── test_cost_model.py
```

---

## How to Run

### Setup

```bash
git clone https://github.com/ShreyasRiddle/etf-signal-research.git
cd etf-signal-research
pip install -r requirements.txt
```

### Configuration

Create `config/settings.yaml`:

```yaml
polygon:
  api_key: "YOUR_POLYGON_API_KEY"

universe:
  etfs: ["SPY", "QQQ", "IWM"]
  start_date: "2018-01-01"
  end_date: "2024-12-31"

signal:
  gap_threshold: 0.003        # 0.3% minimum gap
  holding_period_minutes: 30
  entry_delay_minutes: 5      # skip opening auction
  vwap_confirm_threshold: 0.001

regime:
  vix_low: 18
  vix_high: 25

costs:
  slippage_bps: 5
  commission: 0.0
  borrow_cost_annual: 0.01    # 1% for IWM shorts
```

### Running

```bash
# Full backtest with all signals and filters
python -m src.backtesting.backtester --config config/settings.yaml

# Walk-forward validation
python -m src.backtesting.walk_forward --config config/settings.yaml --window 24 --step 6

# Capacity analysis
python -m src.backtesting.capacity_analysis --config config/settings.yaml --capital-range 500000 25000000

# Generate performance report
python -m src.analysis.performance_report --results data/results/
```

---

## Tech Stack

- **Python 3.11**
- **pandas / numpy** — bar manipulation, signal computation, vectorized backtesting
- **scipy** — statistical tests (t-tests for signal significance, bootstrap for Sharpe CI)
- **Polygon.io API** — 1-minute bar data for SPY, QQQ, IWM
- **FRED API** — VIX daily close
- **matplotlib / seaborn** — performance charts, regime plots, decay curves
- **pytest** — unit tests for signal logic, cost model, backtester

---

## Honest Assessment

**What I'm confident about:**
- The gap-momentum signal is real. It's consistent across 7 years, 3 ETFs, and multiple walk-forward windows. The 2-hour decay pattern is too consistent to be noise.
- The VIX regime filter adds genuine value. It avoids the worst drawdowns without over-tuning (only 2 parameters: 18 and 25).
- The capacity limit ($8M) is realistic. It's consistent with the Almgren-Chriss model and with what I'd expect for a short-horizon signal in the most liquid ETFs.

**What I'm less certain about:**
- Whether this signal will persist. Intraday momentum has been documented in academic literature (Gao et al. 2018, "Intraday Momentum"), but the specific gap-up variant may be over-harvested by HFTs over time.
- Whether 5bps slippage is realistic for all conditions. During volatile opens (the first 5 minutes after market open), slippage can be much higher. The 5-minute entry delay helps, but doesn't eliminate this risk entirely.
- Whether the VWAP confirmation filter is overfitted. It improved Sharpe from 1.45 to 1.71, but I only tested it on the same dataset (albeit out-of-sample). More data would help.

**What this project is NOT:**
- A trading system. There's no live execution, no order management, no real-time data feed. It's research.
- A claim of "free money." A 1.45 Sharpe on 30-minute holds in SPY is thin edge that requires discipline, low costs, and favorable market conditions.
- A finished product. This is a snapshot of research as of January 2026. I continue to monitor whether the signal persists.

---

## References

- Gao, L., Han, Y., Li, S.Z., & Zhou, G. (2018). "Market Intraday Momentum." *Journal of Financial Economics*.
- Almgren, R. & Chriss, N. (2000). "Optimal Execution of Portfolio Transactions." *Journal of Risk*.
- Bouchaud, J.P. (2010). "Price Impact." *Encyclopedia of Quantitative Finance*.

---

## License

MIT License. This is research, not financial advice.
