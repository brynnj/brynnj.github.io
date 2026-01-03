---
title: "Orthogonal Asset Allocation with Downside Risk Control"
layout: project
tags: [trading, portfolio, optimization, python]
---

## Overview

This project is an attempt to build a portfolio with orthogonal and inversely correlated assets, with a few designated growth vehicles and some hedge / flat-market protection, while maintaining the best returns possible.

The practical motivation is that if portfolio-level drawdowns are structurally smaller, it becomes feasible to apply **higher leverage** at the portfolio level without taking on the same tail risk.

The system runs as a live (paper) strategy with weekly rebalancing, using the same deployment and monitoring framework as my other trading experiments.

---

## Core Idea

The portfolio is built around two roles:

1. **Growth assets**  
   Assets with strong long-term return profiles that tend to be volatile or regime-dependent.

2. **Protection / diversifiers**  
   Assets that perform better in sideways or risk-off regimes, or that are meaningfully uncorrelated (or inversely correlated) with the growth sleeve.

The intent was a correlation matrix where the growth vehicles are as uncorrelated as possible, with sufficient inverse correlation to be protected against heavy drawdowns. I ended up selecting TQQQ and UGL for growth, and VXZ, SVXY, and RWM for protection.

(You’ll see EDZ instead of RWM in the backtests and the matrix below — I swapped in RWM for EDZ in live testing because I decided EDZ’s backtest performance was only good by chance in the 2022 bear market, and RWM should fill the same role with higher returns in general. The correlation with the rest of the assets is very similar to EDZ.)

![Correlation:](/assets/img/correlation-matrix.png)

---

## Inspiration

The initial idea for this project was inspired by a QuantConnect research post on risk parity with leveraged ETFs:

https://www.quantconnect.com/research/17879/a-risk-parity-approach-to-leveraged-etfs/p1

That work helped frame the core concept of combining growth-oriented assets with defensive components so that portfolio-level risk, rather than individual asset volatility, becomes the design target.

I ended up swapping out/removing some assets, and shifted the optimization objective a bit.

---

## Strategy Structure

The strategy runs on a **weekly rebalance cycle** and produces target portfolio weights by solving a constrained optimization problem on downside variance.

At a high level:
1. Pull historical data for the asset universe.
2. Compute daily returns and clean obvious data issues.
3. Solve a constrained optimization problem to determine target weights.
4. Reconcile current positions against those targets.
5. Submit trades via Alpaca (paper for now).

---

## Weight Selection

Weight selection is a constrained optimization problem over the last year of daily returns.

First, define portfolio daily returns as:

- **rₚ(t) = R(t) · w**

where **R(t)** is the vector of asset returns on day *t*, and **w** is the weight vector.

The objective is downside variance, meaning the optimizer only penalizes negative portfolio days:

- Let **d(t) = min(rₚ(t), 0)**
- Minimize **mean(d(t)²)** over the lookback window

So in practice:
- compute `port_returns = returns @ w`
- zero out the positive days
- take `mean(negative_days**2)`

On top of that objective, the optimizer applies a few constraints to maintain positive returns:

- **Fully invested:** weights sum to 1.
- **Per-asset bounds:** each asset weight is clipped to `[min_weight, max_weight]` (keeps the portfolio from collapsing into a single “winner”).
- **Sleeve allocation band:** a designated subset of tickers (the set of hedging assets that have negative average returns) must stay within a bounded total allocation:
  - `min_neg_ret ≤ sum(w[sleeve]) ≤ max_neg_ret`

That sleeve constraint is there to keep the optimizer from building a portfolio that stays flat. I tried it without that constraint for a while, and the minimum downside-variance solution will inherently minimize upside variance as well, producing an exceptionally stable portfolio that never grows.

Tuning this band goes a long way toward controlling portfolio performance, but it effectively acts as a knob for desired risk posture.

The problem is solved with **SLSQP**, starting from equal weights.

Performance does not appear to be especially sensitive to the exact optimization; this could likely be accomplished nearly as well with fixed allocations.

---

## Results

A few high-level observations:
- Live behavior has stayed within the family of backtest outcomes, despite some differences in constraint logic.
- The portfolio underperformed pure tech and gold over the most recent period, which isn’t surprising given how strong those assets were individually.
- Drawdowns were materially smaller than most single-asset alternatives.

That tradeoff — smoother equity at the cost of lagging during extreme single-asset runs — is consistent with the intent of the strategy.

You’ll notice the optimization problem isn’t doing much in the way of dynamically changing weights. Within the limits set on risk-on vs hedging asset classes, it tends to pick fairly constant ratios aside from a few events. As mentioned above, this could probably be implemented as a fixed-weight portfolio.

**[Figure: Rolling portfolio weights in live testing]**  
![Weights:](/assets/img/weight-allocations.png)

**[Figure: Portfolio equity curve in live testing]**

The unusual relationship between tech and gold recently (both growing astronomically) led to more correlation and a higher drawdown than I’d like in recent months.

As for backtests, I got rate-limited by yfinance after pulling too much data in a single day and need to fix my backtesting pipeline. I do have an example with generally similar allocations over time, although with much looser bounds (you’ll see much wider swings in allocations).

I also included a rough tax model of ~10% annual tax on gains, but I haven’t yet determined the correct realized tax burden. The cumulative value shown does **not** have taxes subtracted. As noted above in the correlation matrix, the backtests use EDZ, but I substituted RWM for live testing.

**[Figure: Rolling portfolio weights in backtesting]**  
![Backtest Weights:](/assets/img/allocations-backtest.png)

**[Figure: Portfolio equity curve in backtesting]**  
![Backtest Performance:](/assets/img/risk-parity-performance-backtest.png)

---

## Live Testing & Infrastructure

This strategy runs inside the same live framework as my other deployments, including the AI-driven ETH trader.

Shared components include:
- data fetching and caching,
- strategy state tracking,
- performance logging,
- and deployment orchestration.

The live (paper) instance runs on an AWS EC2 machine alongside other experiments, which makes it easier to compare behavior across very different strategy types using the same tooling.

---

## Limitations

- Asset selection is still manual and opinionated.
- Backtesting infrastructure needs cleanup after hitting rate limits on historical data pulls.
- Optimization depends on lookback window and universe choice.
- Results shown are paper trading only; live execution would require additional safeguards.

---

## Next Steps

1. **Fix the backtesting pipeline**  
   The current setup relied too heavily on yfinance, and I was restricted after pulling too much data on a different project. I need to port this to a more robust data provider. I also want to explicitly test this optimization problem against others — the original version of this project was risk parity, which is what the backtest shown uses, and I’d like to see whether that actually performed better over the live testing period. I also need to test explicitly with the RWM substitution for EDZ.

2. **Experiment with synthetic assets**  
   I want to generate synthetic return streams with controlled correlation structure to see if I can improve returns in the orthogonal or hedging assets.

3. **Model taxes explicitly in backtests**  
   Weekly rebalancing requires selling, and it’s not yet clear how much tax drag this introduces. It should be much less than taxing all gains with correct lot management, but still nonzero. Properly modeling realized gains and tax treatment is necessary to understand net performance.

4. **Optimize leverage and test with margin costs**  
   This would better reflect how the strategy would actually be deployed. I’d like to try applying the Kelly criterion here and see whether it produces a better optimal return than SPY or other baselines using this structure.
