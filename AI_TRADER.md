---
title: "AI Trader: LLM-Driven ETH Perps"
layout: project
tags: [trading, llm, crypto, python, automation]
---

# AI Trader: LLM-Driven ETH Perps

## Overview
An hourly crypto trading system that uses an LLM to generate trade decisions for ETH perpetuals, with strict schema enforcement, guardrails, and a broker abstraction layer for execution.

## Problem
Manual crypto trading is time-consuming and inconsistent. I wanted a system that can summarize multi-timeframe market context, produce repeatable decisions, and only trade when the signal is strong.

## Approach / Architecture
The system is split into data collection, LLM decisioning, safety validation, and broker execution, with a scheduler coordinating hourly cycles. The design keeps the LLM isolated behind strict output schemas and a safety layer before any order is placed.

## Key technical details
- Snapshot pipeline aggregates 30m price/volume bars, daily indicators, and recent news into a compact prompt.
- JSON schema enforcement using Pydantic ensures the LLM returns structured decisions.
- Safety guard validates actions, risk bounds, and required fields before execution.
- Broker abstraction supports paper and exchange-backed implementations with a shared order interface.
- Strategy cycle skips entries if a position is already open and supports TP/SL enforcement.

## Results
- Placeholder: add live or simulated metrics (trade count, win rate, PnL, drawdown).
- Placeholder: include a short comparison vs a baseline (buy-and-hold or flat).

![Figure: Strategy Flow](assets/ai-trader-flow.png)
Caption: High-level architecture diagram showing data ingestion, LLM decisioning, safety checks, and order execution.
This figure should show the end-to-end pipeline from market data to broker order placement.

![Figure: Trade Timeline](assets/ai-trader-trade-timeline.png)
Caption: Example trade timeline with entries/exits, TP/SL hits, and equity curve.
This figure should show how the system behaved over a sample period and highlight risk controls.

## How to run / reproduce
1. Configure API keys and broker credentials (OpenAI + exchange).
2. Run the hourly cycle script or invoke the strategy runner from the project root.
3. Review logs and performance artifacts written to the strategy output directory.

## Limitations
- Evaluation is still limited; needs a consistent backtest or replay harness.
- LLM decisions are sensitive to prompt context and market regime changes.
- Live execution paths require careful monitoring and safety overrides.

## Next steps
- Add a deterministic replay harness for historical snapshots.
- Publish standardized performance metrics and a reproducible evaluation report.
- Improve execution reliability and add richer telemetry around decisions.

