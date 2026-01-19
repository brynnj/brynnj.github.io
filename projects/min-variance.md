---
title: "Orthogonal Asset Allocation with Downside Risk Control"
layout: project
tags: [trading, portfolio, optimization, python]
---

## Overview

This project is an attempt to build a portfolio with differently correlated assets, with a few designated growth vehicles and some hedge / flat market protection, while maintaining the best returns possible.

The goal is to build a portfolio with decent returns but low drawdowns, so it becomes feasible to apply higher leverage without taking on the same tail risk, hopefully achieving higher risk-adjusted returns as a result.

The system runs as a live (paper for now) strategy with weekly rebalancing, run from an EC2 instance.

---

## Core Idea

The portfolio is built around two roles:

1. **Growth assets**  
   Assets with strong long-term return profiles that tend to be volatile or regime-dependent.

2. **Protection / diversifiers**  
   Assets that perform better in sideways or risk-off regimes, or that are meaningfully uncorrelated (or inversely correlated) with the growth sleeve.

The intent was a correlation matrix where the growth vehicles are as uncorrelated as possible, with sufficient inverse correlation across the rest of the assets to be protected against heavy drawdowns. I ended up selecting TQQQ and UGL for growth, SVXY for another weaker but low correlation growth, and VXZ and RWM for inverse correlation.

(You’ll see EDZ instead of RWM in the backtests and the matrix below — I swapped in RWM for EDZ in live testing because I decided EDZ’s backtest performance was only good by chance in the 2022 bear market, and RWM should fill the same role with higher returns in general. The correlation with the rest of the assets is very similar to EDZ.)

**Correlation Matrix**  
![Correlation:](/assets/img/correlation-matrix.png)

---

## Inspiration

The initial idea for this project was inspired by a QuantConnect research post on risk parity with leveraged ETFs:

https://www.quantconnect.com/research/17879/a-risk-parity-approach-to-leveraged-etfs/p1

That work helped with the core concept of combining growth-oriented assets with defensive components so that portfolio-level risk, rather than individual asset volatility, becomes the design target.

I ended up swapping out/removing some assets, and shifted the optimization objective a bit, although I haven't meaningfully proven that it's a better objective yet.

---

## Strategy Structure

The strategy runs on a weekly rebalance cycle and produces target portfolio weights by solving a constrained optimization problem on downside variance.

At a high level:
1. Fetch historical data for each asset. I cache data, so the actual requests are only for the missing days.
2. Compute daily returns and clean obvious data issues.
3. Solve a constrained optimization problem to determine target weights.
4. Reconcile current positions against those targets (how much to buy and sell of each).
5. Submit trades via Alpaca (paper for now). In practice I'd optimize tax-lot selection for rebalancing, but Alpaca doesn't support that and I'm just paper trading so I haven't implemented that yet.

---

## Weight Selection

Weight selection is a constrained optimization problem over the last year of daily returns.

First, define portfolio daily returns as:

- **rₚ(t) = R(t) · w**

where **R(t)** is the vector of asset returns on day *t*, and **w** is the weight vector.

The objective is downside variance, meaning the optimizer only penalizes negative portfolio days:

- Let **d(t) = min(rₚ(t), 0)**
- Remove the zeros
- Minimize **mean(d(t)²)** over the lookback window

So in practice:
- compute `portfolio_returns = returns @ w`
- zero out the positive days
- take `mean(negative_days**2)`

On top of that objective, the optimizer applies a few constraints to maintain positive returns:

- **Fully invested:** weights sum to 1.
- **Per-asset bounds:** each asset weight is clipped to `[min_weight, max_weight]` (keeps the portfolio from collapsing into a single “winner”).
- **Sleeve allocation band:** a designated subset of tickers (the set of hedging assets that have negative average returns) must stay within a bounded total allocation:
  - `min_neg_ret ≤ sum(w[sleeve]) ≤ max_neg_ret`

That sleeve constraint is there to keep the optimizer from building a portfolio that stays flat. I tried it without that constraint for a while, and the minimum downside-variance solution will inherently minimize upside variance as well, producing an exceptionally stable portfolio that never grows.

Tuning this band goes a long way toward controlling portfolio performance, but it effectively acts as a knob for desired risk posture that has to be picked manually.

The problem is solved with **SLSQP**, starting from equal weights.

Performance does not appear to be especially sensitive to the exact optimization; this could likely be accomplished nearly as well with fixed allocations. I need to do some more testing to see if the optimizations are actually leading to improved performance on the desired metric.

---

## Results

A few high-level observations:
- Live behavior has stayed within the family of backtest outcomes, despite some differences in constraint logic.
- The portfolio underperformed just the growth assets, but that is to be expected (especially given their extremely strong individual performances recently) and the drawdown was materially better.

You’ll notice the rolling window optimization isn’t doing much in the way of dynamically changing weights. Within the limits set on risk-on vs. hedging asset classes, it tends to pick fairly constant ratios aside from a few events. As mentioned above, this could probably be implemented as a fixed-weight portfolio.

**Rolling portfolio weights in live testing**  
![Weights:](/assets/img/weight-allocations.png)

**Portfolio equity curve in live testing**
![Equity curve:](/assets/img/cumulative-returns.png)

**Drawdown in live testing**  
![Live test drawdown:](/assets/img/drawdown_live.png)

The unusual relationship between tech and gold recently (both performing well simultaneously) led to more correlation and a higher drawdown than I’d like in recent months, but still in-family relative to backtesting.

As for backtests, I got rate-limited by yfinance after pulling too much data in a single day (for a different project) and need to fix my backtesting pipeline to use a different source. I do have an example below with generally similar allocations over time, although with much looser bounds (you’ll see much wider swings in allocations).

I also included a rough tax model of ~10% annual tax on gains, but I haven’t yet modeled the correct realized tax burden. The cumulative value shown does **not** have taxes subtracted, but taxes are shown on the chart. As noted above in the correlation matrix, the backtests use EDZ, but I substituted RWM for live testing. 

**Rolling portfolio weights in backtesting**  
![Backtest Weights:](/assets/img/allocations-backtest.png)

**Portfolio equity curve in backtesting**  
![Backtest Performance:](/assets/img/risk-parity-performance-backtest.png)

Drawdowns reached around 12%, higher than anything seen so far in live. I do think the backtest is very optimistic for performance in 2022 and 2023; I think the combination of EDZ with tech did well for no real structural reason. I'd expect drawdowns closer to probably 20-30% over a sustained downtrend like that, but need to backtest with RWM.

**Drawdown in backtesting**  
![Backtest Drawdown:](/assets/img/drawdowns-backtest.png)

---

## Limitations

- Asset selection is still manual and opinionated
- Backtesting infrastructure needs some fixing
- Optimization depends on lookback window and universe choice
- Results shown are paper trading only; live execution would require additional safeguards

---

## Next Steps

1. **Fix the backtesting pipeline**  
   The current setup relied too heavily on yfinance, and I was restricted after pulling too much data on a different project. I need to port this to a more robust data provider. 
   
2. **Study the optimization problem more** 
   I also want to explore other optimization objectives — the original version of this project was risk parity, which is what the backtest shown above uses, and I’d like to see whether that actually performed better over the live testing period. I also need to test explicitly with the RWM substitution for EDZ.

3. **Experiment with synthetic assets**  
   It would be interesting to generate synthetic assets (i.e. a custom strategy) with controlled correlation to my main growth drivers to see if I can improve returns in the orthogonal or hedging assets. For example, a short-focused strategy even with flat or slightly negative returns would be inversely correlated to the market and might fill a good role here.

4. **Model taxes explicitly in backtests**  
   Weekly rebalancing requires selling, and it’s not yet clear how much tax drag this introduces. It should be much less than taxing all gains with correct lot management, but still nonzero. Properly modeling realized gains and tax treatment is necessary to understand net performance.

5. **Optimize leverage and test with margin costs**  
   This would better reflect how the strategy would actually be deployed. I’d like to try applying the Kelly criterion here and see whether it produces a better optimal return than SPY or other baselines using this structure.
