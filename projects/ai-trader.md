---
title: "AI Trader: LLM-Driven ETH Perps"
layout: project
tags: [trading, llm, crypto, python, automation]
---

## Overview

This is an experimental crypto trading system that uses an LLM to generate trade decisions for ETH perpetual futures.

The core idea was to see what happens if you give an LLM a compact snapshot of market context and ask it to make a concrete trading decision — entry, direction, sizing, and exits — under strict constraints.

The system runs on a fixed **30-minute cycle** and trades relatively infrequently by design. Everything routes through a safety layer before anything touches an exchange.

---

## Why I Built It

Ethereum, and a handful of other crypto assets, sit in an interesting middle ground:
- liquid enough that slippage is manageable,
- volatile enough that there’s something to trade,
- and still heavily driven by retail participation and sentiment.

Just by observation of a price history and anecdotal experience from others I've talked to, crypto exhibits very clean tradeable momentum at time that a human could likely capitalize on given broader market context, if they could sit at a computer 24/7.

I wasn’t expecting an LLM to predict price. The experiment was more about whether it could reason when given good context and behave more like a discretionary trader than a fixed rule set.

One practical constraint here is cost: I built this while working around **free / low-cost ChatGPT usage limits**, which meant prompts had to stay concise and decision frequency had to be limited. Running on a 30-minute cadence was a natural fit for that constraint, and also helped keep the problem well-posed.

ETH perps were a convenient testbed: high liquidity, simple mechanics, and at the time, very low fees on Coinbase, which made it feasible to run live without fees dominating the outcome.

---

## High-Level Architecture

<!-- PLACEHOLDER: system flow diagram -->
**[Figure: AI trader system flow]**  
*Data snapshot → LLM decision → safety validation → broker execution*

The system is split into a few clean stages:

1. **Data collection** builds a compact snapshot of recent market state.
2. **LLM decisioning** produces a structured trade decision.
3. **Safety validation** checks the decision against hard constraints.
4. **Execution** places or manages orders via a broker abstraction.
5. A scheduler coordinates the whole cycle on a fixed 30-minute cadence.

The LLM is intentionally isolated behind schemas and validation logic. It never talks to an exchange directly.

---

## Data & Prompting

Each decision cycle builds a snapshot that includes:
- recent price and volume bars (30m),
- higher-timeframe context (daily indicators),
- and a short summary of recent relevant news (including adjacent news like BTC and general market sentiment, to capture the risk-on or risk-off feel).

The snapshot has to stay compact — both to keep prompts affordable and to force prioritization of what actually matters. That constraint shaped the entire design more than I expected.

Over time, the biggest changes weren’t architectural — they were in:
- how compressed the snapshot was,
- what signals were emphasized or removed,
- and how explicitly the prompt constrained risk and behavior.

Small wording changes reliably shifted the system between trend-following and mean-reversion behavior.

---

## Decision & Safety Layer

The LLM is required to return a JSON object that conforms to a strict schema:
- action (enter / hold / exit),
- direction,
- sizing,
- stop loss and take profit levels,
- and reasoning text (for logging only).

Pydantic validates this output before anything happens.

On top of that, a safety layer enforces:
- position limits,
- required SL/TP presence,
- no new entries if a position is already open,
- and general sanity checks.

If anything fails validation, the cycle is skipped.

---

## Execution

Execution runs through a broker abstraction so the same logic supports:
- paper trading,
- and live exchange-backed trading.

Order placement, position tracking, and exits are handled outside the LLM loop. The model only decides *what* it wants to do, not *how* orders are executed.

All of this was deployed on an AWS EC2 instance that I spun up for myself a year ago. I have a framework reasonably well abstracted for these deployments, so this shared a good chunk of code with the min variance deployment on Alpaca.

---

## What Actually Happened

Behavior depended heavily on market regime and prompt bias.

Early on, I paper traded the system during a strong momentum phase. The prompts were momentum-oriented, and performance looked very good.

When I switched to live trading, ETH moved into a long, choppy sideways period with frequent false breakouts. In that regime:
- the system hovered around breakeven for months,
- then started giving back money once volatility picked up in the wrong way.

At that point, I disabled it.

Over the full tested period, it did outperform holding ETH outright and stayed out of major downtrends. But the drawdowns during regime shifts were large enough that I wasn’t comfortable leaving it running unattended.

<!-- PLACEHOLDER: trade timeline / equity curve -->
**[Figure: Example trade timeline and equity curve]**

---

## Takeaways So Far

A few things stood out:
- Prompt wording has outsized influence on behavior.
- Without explicit regime awareness, the system drifts toward a single bias.
- The model is better at explaining *why* it wants to act than at knowing when its assumptions have stopped holding.

One thing I haven’t implemented yet — but plan to — is feeding recent trade performance back into the snapshot in a very compact form. Even minimal self-feedback might help the system adjust across regimes.

---

## Limitations

- Evaluation so far has been mostly live and forward-looking; there’s no full historical replay harness yet.
- Behavior is sensitive to both prompt phrasing and market regime.
- Execution always requires conservative safeguards and monitoring.

---

## Where This Is Going

The most important next step would be to figure out a way to backtest without blowing through tokens. Not sure yet how to pull that off.

Longer term, this project is probably heading into more traditional model-based or RL-driven approaches using a similar feature set. The LLM doesn't appear to be adding all that much in the way of interpreting price and volume action, so it's just becoming a feature engineering exercise, and the other models remove all the token use constraints holding me back. (See the ETH metalabeling project if I've gotten around to posting it here yet)

For now, it’s a useful sandbox for building a system around LLM reasoning, safeguarding it well enough to actually deploy my own money live, and building a robust live pipeline that ran and logged for months with successfully handled errors.
