---
title: "AI Trader: LLM-Driven ETH Perps"
layout: project
tags: [trading, llm, crypto, python, automation]
---

# AI Trader: LLM-Driven ETH Perps

## Overview
An hourly crypto trading system that uses an LLM to generate trade decisions for
ETH perpetuals, with strict schema enforcement, guardrails, and a broker
abstraction layer for execution.

## Problem
Ethereum and some of the other relatively liquid but volatile coins out there seem to have patterns that an experienced discretionary trader could exploit. However, a person can't watch the markets 24/7. I wanted to see if I could get an LLM to take in 
multi-timeframe market context and news, and produce decisions with more nuanced reasoning than what a simple rules-based or ML system could do.

Obviously I didn't expect an LLM to magically predict price - the art is in how market context is provided and the prompting language, both of which need some work in my current revision. One could argue that at that point you might as well just train a model or build an RL system with the same feature set - I would agree, and this is my next project. This project was just for fun, but actually had surprisingly good results given the relatively simple implementation I tried.

I opted to trade on ETH perp futures - liquid enough to have small slippage, volatile enough to make significant gains, hopefully not a perfectly efficient market given large retail participation, but most importantly had a promotion on coinbase where fees were negligibly small so I could test this without burning too much money.

## Approach / Architecture
The system is split into data collection, LLM decisioning, safety validation,
and broker execution, with a scheduler coordinating half-hourly cycles. The design
keeps the LLM isolated behind strict output schemas and a safety layer before
any order is placed.

## Key technical details
- Snapshot pipeline aggregates 30m price/volume bars, daily indicators, and
  recent news into a compact prompt.
- JSON schema enforcement using Pydantic ensures the LLM returns structured
  decisions.
- Safety guard validates actions, risk bounds, and required fields before
  execution.
- Broker abstraction supports paper and exchange-backed implementations with a
  shared order interface.
- Strategy cycle skips entries if a position is already open and supports TP/SL
  enforcement.

## Results
- Unsurprisingly, this still ends up either falling into a trend-following or mean-reversion biased approach depending on prompt wording, and as such bleeds a bit too much when in the wrong regime. I initially paper traded this in a strong momentum regime, and my prompting was always biased towards momentum, so it did exceptionally well. When I went in with real money, ETH started going mostly sideways with lots of false breakouts. I was just under breakeven for months, but some recent volatility started losing a concerning amount of money so I've disabled it for now.

Over the full tested timeframe, this bot actually did significantly outperform the underlying asset. It stayed out in the major downtrends. I think there is actually potential here if I can expand the feature set to enable better regime classification. I also think adding a "reinforcement" element would go a long way - if I can find a way to concisely provide recent trade performance alongside market context, I think the LLM could infer regimes accordingly.


<figure>
  <img src="/assets/img/ai-trader-flow.png" alt="AI Trader system flow diagram">
  <figcaption>
    High-level architecture diagram showing data ingestion, LLM decisioning,
    safety checks, and order execution.
  </figcaption>
</figure>

<figure>
  <img src="/assets/img/ai-trader-trade-timeline.png" alt="AI Trader example trade timeline">
  <figcaption>
    Example trade timeline with entries/exits, TP/SL hits, and equity curve.
  </figcaption>
</figure>

## How to run / reproduce
1. Configure API keys and broker credentials (OpenAI + exchange).
2. Run the hourly cycle script or invoke the strategy runner from the project
   root.
3. Review logs and performance artifacts written to the strategy output
   directory.

## Limitations
- Evaluation is still limited; needs a consistent backtest or replay harness.
- LLM decisions are sensitive to prompt context and market regime changes.
- Live execution paths require careful monitoring and safety overrides.

## Next steps
- Add a deterministic replay harness for historical snapshots.
- Publish standardized performance metrics and a reproducible evaluation report.
- Improve execution reliability and add richer telemetry around decisions.
