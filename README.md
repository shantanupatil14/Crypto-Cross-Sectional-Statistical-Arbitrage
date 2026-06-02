# Crypto Momentum-Reversal Strategy
### A cross-sectional long/short research journey — from naive patterns to production-grade alpha

![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![Status](https://img.shields.io/badge/Status-Research-orange)

---

## Overview

This repository documents a complete quantitative research journey building a
**dollar-neutral, market-neutral, cross-sectional long/short strategy** on 50
liquid crypto USDT pairs on Binance.

The goal was to answer a deceptively simple question: can a systematic strategy
generate alpha from the crypto asset class without taking directional exposure
to Bitcoin?

**The answer is yes — but it took seven trials, three of which failed badly, to
get there.** Every failure is documented with a diagnosis. Every design decision
is explained. No results are invented or cherry-picked.

This project does not use any machine learning models. Every signal and
every position is traceable to an economic mechanism, a mathematical
formulation, and a documented reason for inclusion.

---

## Final Results — Trial 7 / V2.0

| Metric | In-Sample (2021–2023) | Out-of-Sample (2024–2025) |
|---|---|---|
| CAGR | 12.88% | **26.06%** |
| Sharpe Ratio | 0.73 | **1.23** |
| Sortino Ratio | 1.05 | **2.38** |
| Max Drawdown | −19.58% | −22.58% |
| Beta to BTC | 0.01 | **−0.03** |
| Annualized Alpha | 13.68% | **26.71%** |
| Avg Daily Turnover | 19.62% | 21.93% |

**Stress test:** OOS Sharpe remains **0.84** at 2× transaction costs (20bps).  
**Parameter stability:** 9-combination optimizer sweep produces Sharpe range 1.06–1.36.

---

## Repository Structure

```
crypto-momentum-reversal/
│
├── notebooks/
│   └── Final_Crypto_Momentum_Reversal.ipynb   # Full research journey (Trials 1–7)
│
├── data/
│   └── data_store_1/                          # Local only — not committed (see below)
│       └── {TICKER}-1h.pkl                    # One file per ticker, hourly OHLCV
│
├── src/                                       # Modular source (extracted from notebook)
│   ├── data_loader.py                         # load_price_data() — 23:00 UTC bar logic
│   ├── signals.py                             # balanced_hedge_fund_signals_v2()
│   ├── optimizer.py                           # hf_optimizer_mv_v2() — MV + ADTV + beta
│   ├── backtest.py                            # run_production_system_v2_enhanced()
│   └── utils.py                              # calc_stats(), shared indicators
│
├── results/
│   └── trial_summary.md                       # Known results table for all 7 trials
│
├── .gitignore
├── requirements.txt
└── README.md
```

---

## Universe

50 USDT pairs on Binance spanning large-cap, mid-cap, and select DeFi/L1 tokens:

```
BTCUSDT  ETHUSDT  SOLUSDT  BNBUSDT  ADAUSDT  XRPUSDT  DOGEUSDT
LTCUSDT  LINKUSDT BCHUSDT  MATICUSDT DOTUSDT TRXUSDT  SHIBUSDT
AVAXUSDT XLMUSDT  UNIUSDT  ATOMUSDT  ETCUSDT HBARUSDT ICPUSDT
FILUSDT  LDOUSDT  APTUSDT  NEARUSDT  ARBUSDT OPUSDT   MKRUSDT
AAVEUSDT GRTUSDT  STXUSDT  ALGOUSDT  RNDRUSDT EGLDUSDT FLOWUSDT
THETAUSDT QNTUSDT SANDUSDT MANAUSDT  AXSUSDT CHZUSDT  NEOUSDT
EOSUSDT  IOTAUSDT KAVAUSDT ZECUSDT   DASHUSDT FTMUSDT  GALAUSDT CRVUSDT
```

BTC is used as **benchmark and beta reference only** — it receives zero active weight.

---

## Data Pipeline

- **Source:** Binance Vision 1-hour OHLCV (2021–2024) + Yahoo Finance (2025 YTD)
- **Decision bar:** 23:00 UTC close — the last complete hourly bar before midnight rebalancing
- **Why not daily resample:** `resample('D').last()` is time-ambiguous. The 23:00 bar is exact.
- **Volume:** 24-hour rolling sum ending at 23:00 UTC, used for ADTV cap calculations
- **Data not committed** to this repo due to file size — see [Data Setup](#data-setup) below

---

## Strategy Architecture

### Signal — V2.0

```
# Short-term reversal (3–5 days)
reversal = −0.6 × ret_3d − 0.4 × ret_5d

# Medium-term momentum (7–21 days) with volume agreement filter
vol_multiplier = 1 + tanh(vol_z)              # 0→2, suppresses thin-volume moves
momentum = (0.3×mom_7d + 0.4×mom_14d + 0.3×mom_21d) × vol_multiplier

# Adaptive regime blend
regime_weight = −0.5×vol_zscore + 0.5×trend_consistency   # ∈ [−1, +1]
adj_rev = 0.55 − 0.15 × regime_weight         # more reversal when choppy
adj_mom = 0.35 + 0.15 × regime_weight         # more momentum when trending

# Combined → 60d z-score → lag(1) → cross-sectional rank → demean → normalize
raw = adj_rev×reversal + adj_mom×momentum + 0.10×trend_strength×100
```

### Optimizer — V2.0

```
Maximize:  (signal @ w)  −  (λ_risk/2) · w'Σw  −  λ_tc · ‖w − w_prev‖₁

Subject to:
    Σwᵢ = 0                                    # Dollar-neutral
    ‖w‖₁ ≤ 1                                   # Gross exposure ≤ 1×
    |wᵢ| ≤ min(18%, ADTV_cap)                  # Liquidity-aware position cap
    |β_btc · w| ≤ 0.10                         # Hard BTC beta guardrail

Parameters:  λ_risk = 5.0  |  λ_tc = 0.002
Risk model:  Ledoit-Wolf shrinkage, 60-day rolling window
Solver:      SCS (always available with cvxpy)
Fallback:    hold w_prev (PM-realistic — never guess on solver failure)
```

### Volatility Scaling

```
vol_scalar = clip(target_vol / realized_21d_vol, 0.6, 1.4)
target_full = w_signal × vol_scalar      # scaling applied BEFORE turnover calc
turnover    = |target_full − drifted|    # costs charged on scaled weights
```

---

## Research Journey — Trial Summary

| Trial | Signal | Key Change | IS Sharpe | OOS Sharpe | Outcome |
|---|---|---|---|---|---|
| 1 | Hurst-blended dual regime | Baseline | 1.11 | 0.50 | IS/OOS gap — passive optimizer |
| 2 | AlphaEngine (sign bug) | Continuous z-scores | 0.01 | N/A | Sign flip + 67× gross leverage |
| 3 | AlphaEngine (sign bug) | Utility maximization | −1.59 | −0.10 | −52% IS CAGR, 31.6% daily TO |
| 4 | Signal Logic 3 (reversal) | SCS optimizer | 0.41 | 1.22 | OOS > 1.0 first time; IS weak |
| 5 | Balanced mom+rev | Corrected optimizer | 0.60 | 0.98 | Honest baseline — phantom alpha removed |
| 6 | V2.0 + volume agreement | Volume filter | — | 1.23 | +5.7pp OOS CAGR vs Trial 5 |
| **7** | **V2.0 + volume + ADTV** | **Full production** | **0.73** | **1.23** | **Final system** |

---

## Installation

```bash
git clone https://github.com/YOUR_USERNAME/crypto-momentum-reversal.git
cd crypto-momentum-reversal
pip install -r requirements.txt
```

### requirements.txt

```
pandas>=2.0
numpy>=1.24
cvxpy>=1.3
scikit-learn>=1.3
scipy>=1.11
matplotlib>=3.7
requests>=2.31
yfinance>=0.2.36
python-dateutil>=2.8
```

---

## Data Setup

The `data_store_1/` directory is excluded from this repository (see `.gitignore`).

To build the data locally:

1. Open the notebook and run **Section 1.5 — Data Pull** (marked `OPTIONAL — SKIP IF ALREADY RUN`)
2. The cell downloads monthly Binance Vision ZIP files for each ticker from 2021 onward
3. Yahoo Finance supplements 2025 data where Binance Vision hasn't published yet
4. All files are saved as `data_store_1/{TICKER}-1h.pkl`

Approximate total size: ~2GB for all 50 tickers at hourly granularity 2021–2025.

---

## Running the Notebook

```bash
jupyter notebook notebooks/Final_Crypto_Momentum_Reversal.ipynb
```

All heavy backtest cells are safe by default — they are commented with
`# Uncomment to run`. Saved results print immediately without re-execution.

To run a specific trial end-to-end, uncomment only that trial's backtest cell.
Trials 1–7 are independent and can be run in isolation once `prices`, `volumes`,
`returns`, and `btc_ret` are loaded (Section 1).

---

## Known Limitations

1. **Single chronological split** — one IS/OOS boundary is one data point. Walk-forward
   or combinatorial purged cross-validation (CPCV) would provide stronger robustness evidence.

2. **Fixed universe** — the 50-coin list was chosen with knowledge of which coins survived
   to 2025. Coins that delisted or collapsed (LUNA, FTT) are not included, introducing
   survivorship bias.

3. **Cost model simplification** — 10bps linear cost is a reasonable assumption for
   large-cap pairs but likely understates market impact for smaller-cap positions
   at the modeled $10M AUM.

4. **No funding rate signal** — perpetual futures funding rates are a well-documented
   predictor of crypto reversals and are not yet incorporated.

---

## What's Next

- [ ] Walk-forward validation / CPCV
- [ ] 4-hour rebalancing (hourly data already available)
- [ ] Funding rate signal integration
- [ ] Sector-relative signals (DeFi vs L1 vs gaming)
- [ ] Live paper trading via Binance testnet

---

> **DISCLAIMER:** This repository is for educational and research purposes only.
> Nothing in this repository — including the strategy, results, code, analysis,
> commentary, or any other content — constitutes investment advice, financial advice,
> trading advice, or a recommendation to buy or sell any financial instrument or
> asset. Past performance of a backtested strategy does not guarantee future results.
> All backtested results are hypothetical and subject to significant limitations
> including but not limited to look-ahead bias, survivorship bias, and overfitting.
> The author is not a registered investment advisor. Do not make any investment
> decision based on content in this repository. This is not an investment opinion.
