---
layout: project
title: "Full Algorithmic Trading Platform"
description: "A modular, event-driven day trading platform built in Python with async data streams, dynamic stock selection, backtesting, live paper trading, and a real-time UI stack (PyQtGraph + optional C++/Qt visualizer). It focuses on end-to-end pipeline design: scanning, data ingestion, strategy orchestration, execution, and observability."
year: 2025
tags: [trading, python, asyncio, data-pipeline, backtesting, ui, systems]
---

## Overview
This project is my highest-effort and most complete one: a full-stack automated trading system built to run live and to be testable offline. The focus is on modular architecture and robustness more than any single strategy. It combines scanners, streaming market data, strategy state machines, order execution, position tracking, and a real-time GUI, with tooling for backtests and synthetic data generation.

![Figure 1: End-to-end system architecture](assets/img/trading-platform-architecture.png)
Caption: Figure 1. End-to-end data pipeline and async task graph.
This figure should show the scanner feeding a strategy manager, async data streams (Alpaca/Polygon and fake streams), state machines, order executor, position tracker, logs, and UI components with arrows for real-time flow.

## Problem
I wanted a real trading sandbox that makes it easy to:
- scan markets for candidates, not just trade a fixed list,
- run multiple strategies concurrently without blocking,
- switch between live streams and test data without rewriting code, so I can test on the exact code that'll run with real money (learned this one from flight software),
- observe and debug live behavior through a GUI
## Approach / Architecture
The architecture is modular and intentionally layered so each piece can be swapped or tested in isolation.

**Core flow (scanner -> manager -> stream -> state machine):**
1. A StockListManager produces a dynamic set of tickers to trade. This can be a constant list for testing, or a scanner that evaluates live market conditions (price, volume, float , SIC codes, sentiment-based or news-based catalysts).
2. The Strategy Manager polls the stock list. For every added ticker, it spins up a dedicated data stream and registers one or more strategy state machines. For removals, it gracefully tears down tasks.
3. Each ticker's data stream runs in its own async task. It emits bars/quotes to its registered strategies, which are isolated state machines (no shared global state).
4. Each state machine encapsulates trading logic, indicator updates, and visualization hooks, and can signal trades to the execution layer.

This pipeline is the backbone: it is fully asynchronous, per-ticker isolated, and tolerant of individual faults.

![Figure 2: Live UI and control panel](assets/img/trading-platform-ui.png)
Caption: Figure 2. Live charting UI and visualizer control panel.
This figure should show the PyQtGraph visualizer with candlesticks, volume, indicators, and a control panel for toggling strategy visualizers per ticker.

## Key technical details

### Modular data -> manager -> state machine flow (detailed)
**Stock selection layer**
- The `StockListManager` interface defines `update()` and `get_active_stock_list()` so the rest of the system doesn't care how scanning works.
- There are multiple concrete implementations:
  - `ConstantStockListManager` for deterministic tests and backtests.
  - `TopGainerStockListManager` for API-based scanning with sentiment gating.
  - `PolygonTopGainerStockListManager` for a richer pipeline (premarket change %, rVOL, absolute volume, float proxy, SIC exclusions, optional catalyst detection).
- The scanner caches enrichments per ticker so repeated scans are faster and cheaper.

**Strategy orchestration**
- `StrategyManager` is the orchestrator: it polls the stock list, compares deltas (added/removed), and manages per-ticker tasks.
- Each ticker is mapped to:
  - a data stream client,
  - a list of strategies/state machines,
  - an async task that runs the stream.
- The strategy manager handles lifecycle (spawn, cancel, cleanup) and enforces trading constraints (e.g., only one active trade if configured).

**Data streaming**
- The data stream layer is pluggable:
  - `AlpacaWebSocketClient` for live bars/quotes, with warm-start history push.
  - `FakeWebSocketClient` for CSV replay in backtests.
  - `AsyncPolygonAggClient` for robust live data with strict ordering and warm-start snapshots.
- Each stream has a single job: transform data into a normalized schema and push it to subscribed strategies.

**State machines**
- Each strategy is a state machine with explicit handlers for new bars/quotes.
- They receive normalized data from the stream (so input format is consistent across live vs backtest).
- State machines own their own indicators, risk logic, and UI hooks.
- Wherever possible, indicator logic is abstracted out and shared across strategies.
- Even though I'm still running python, everything is pure numpy and as optimized as reasonably possible.
- This design keeps strategy logic pure and testable, while the managers and streamers handle orchestration.

**Execution & positions**
- Order execution and position tracking are abstracted into separate components so strategies don�t speak directly to broker APIs.
- The executor manages orders, timeouts, and cancellations.
- The position tracker is event-driven using account WebSockets for near-real-time updates.

**Hotkey trading system**
- I also built a manual trading system around hotkey-driven execution. This integrates the order executor and position tracker for rapid manual entries and exits, while still running inside the same market data and visualization stack. Some brokerages already offer hotkey trading, but full customization and minimal overhead between key and execution made this worthwhile.

**Visualization**
- A PyQtGraph visualizer renders candlesticks, volume, and indicators per ticker.
- A control panel can dynamically toggle visualizers for active strategies.
- A separate C++/Qt visualizer exists for performance testing and extensibility.

### Data pipeline robustness
- Live streams warm-start with historical bars to seed indicators before live data begins.
- The Polygon client dedupes overlaps and strictly orders 1s and 1m bars to prevent logic drift.
- A single normalized bar/quote schema flows across live, backtest, and synthetic data.

### Testing and simulation
- Backtesting takes existing datasets and replays them at zero delay to stress the state machines and evaluate strategy logic.
- Integrated tests support live paper trading with full orchestration (scanner + strategies + streams + UI).
- For some promising strategies I've built large-scale backtests with Monte Carlo perturbation tests (dispersed slippage, latency, spread, and random missed entries).
- I also build a Databento pipeline to fetch datasets for backtesting and repackage them into the correct schema, so they work with the csv feeds.

![Figure 3: Backtest and log artifacts](assets/img/trading-platform-backtest.png)
Caption: Figure 3. Backtesting outputs and log-based analytics.
This figure should show the backtest log directory and a sample performance table/plot derived from log processing.

[Video 1: Live run walkthrough](assets/video/trading-platform-live-demo.mp4)
Caption: Video 1. Live scan-to-trade demo with UI updates.
This video should show the scanner selecting tickers, strategies spinning up, and the UI updating in real time (paper trading only).

## Examples and Results
- TODO

## Limitations
- Relies on external data providers (Alpaca, Polygon, FMP), so uptime and rate limits are a real constraint. Generally I'm doing my best to get free data which means jumping through a lot of hoops and standardizing across many different formats.
- Strategy logic is still evolving, and the system is not yet hardened for full production risk controls.
- Logic for multiple data feeds in live testing is still buggy and needs work, but currently unclear how much of this is a bad/inconsistent data provider or on my end.

## Next steps
- Figure out the right way to do concurrency in live (async, threading, MP, something else?)
- Build a results dashboard that reads log output and generates standardized performance reports.
- Add CI tests for data integrity and strategy lifecycle events (start/stop, graceful shutdown).
- Expand observability with structured logging and basic runtime metrics (latency, dropped bars).
