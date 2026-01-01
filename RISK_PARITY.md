---
title: "Risk Parity: Weekly Rebalance (Alpaca Paper)"
layout: project
tags: [trading, portfolio, optimization, python]
---

# Risk Parity: Weekly Rebalance (Alpaca Paper)

## Overview
A weekly rebalancing strategy that uses historical covariances to compute a downside-variance-optimized portfolio and executes trades via the Alpaca API in paper mode.

## Problem
I wanted an automated way to manage a diversified ETF portfolio with explicit risk constraints, rather than manually rebalancing or relying on static allocations.

## Approach / Architecture
The strategy pulls historical OHLCV data, computes returns, sanitizes outliers, and solves an optimization problem to produce target weights. It then reconciles positions and submits market orders through Alpaca.

## Key technical details
- Weekly rebalance window with market-day checks and a guard to avoid over-trading.
- Downside-variance optimization with bounds on negative-return assets.
- Unified BaseStrategy framework for logging state, performance, and status.
- Alpaca paper trading integration with order submission and fill checks.

## Results
- Placeholder: add performance summary (CAGR, volatility, drawdown).
- Placeholder: include baseline comparison (e.g., SPY or equal-weight).

![Figure: Allocation History](assets/risk-parity-weights.png)
Caption: Rolling target weights over time with rebalance markers.
This figure should show how the optimizer shifts weights across assets across market regimes.

![Figure: Equity Curve](assets/risk-parity-equity.png)
Caption: Portfolio equity curve versus benchmark.
This figure should show the strategy performance relative to a simple baseline.

## How to run / reproduce
1. Configure Alpaca credentials for paper trading.
2. Run the strategy via the registry runner with the rebalance action.
3. Inspect performance CSVs and logs saved in the strategy output directory.

## Limitations
- Results are not yet backed by a full, reproducible backtest report.
- Optimization is sensitive to lookback window and asset universe choices.
- Paper trading only; live execution needs extra safeguards.

## Next steps
- Add a formal backtest harness and exportable performance report.
- Expand the asset universe and stress-test risk constraints.
- Add parameter sweep tooling to quantify sensitivity.

