---
layout: project
title: "Algorithmic Trading Platform (Research & Experimentation)"
description: "A modular, event-driven trading platform built to explore strategy design, market structure, and live systems behavior using a shared codebase for simulation and paper trading."
year: 2025
tags: [trading, python, asyncio, systems, market-structure]
---

## Overview

This is a full-stack algorithmic trading platform I’ve been building to experiment with trading strategies and market structure in a realistic environment.

The original goal was pretty simple: I wanted something that could run live, replay historical data, and show me what was going on without constantly rewriting or branching code.

Over time, that pushed the design toward:
- shared code paths for live and replay,
- per-symbol isolation,
- and enough visibility that I can tell *why* something is happening, not just that it happened.

---

## Scope and Focus

Most of the experimentation is around **high-volatility, low-float equities in the premarket**.

That’s not because the platform is limited to that regime—it isn’t—but because this space is useful for testing ideas quickly. Symbols come and go, liquidity changes fast, and execution details matter more than they do in slower markets.

The same system could be pointed at other asset classes or timeframes with minimal changes.

---

## High-Level Architecture

<!-- PLACEHOLDER: System architecture diagram -->
**[Figure: End-to-end system architecture]**  
*Scanner → Strategy Manager → Async Data Streams → Strategy State Machines → Execution / UI*

At a high level, the platform is organized as a pipeline:

1. A stock selection layer produces a dynamic list of symbols.
2. A strategy manager reacts to changes in that list.
3. Each symbol runs in its own async data stream.
4. One or more strategy state machines consume normalized market data.
5. Trade signals flow into an execution and position layer.
6. Everything is mirrored into logs and a live UI.

Each symbol is isolated from the others. That decision simplified a lot of things later on.

---

## Core Components

### Stock Selection

Stock selection lives behind a small interface:
- `update()`
- `get_active_stock_list()`

I’ve ended up with a few different implementations depending on what I’m doing:
- fixed symbol lists for testing,
- API-based top gainer scanners,
- premarket scanners with filters on price change, volume proxies, float estimates, and exclusions.

Enrichment data is cached per symbol so repeated scans don’t keep redoing the same work.

---

### Strategy Management

The strategy manager handles lifecycle stuff:
- detecting when symbols are added or removed,
- starting and stopping per-symbol async tasks,
- attaching one or more strategies to each symbol.

Each active symbol maps to:
- one data stream,
- a list of strategy state machines,
- and an async task that routes events.

This structure lets a lot of symbols run concurrently without shared state, which turned out to be important pretty quickly.

---

### Market Data Streams

Market data streams are intentionally simple.

I have different clients for:
- live WebSocket data,
- CSV replay for backtests,
- and aggregation-based streams for higher-fidelity research data.

Everything gets normalized early into the same bar and quote schema, so strategies don’t care where the data came from.

---

### Strategy State Machines

Strategies are implemented as explicit state machines with handlers for new bars and quotes.

Each one:
- owns its indicators and rolling buffers,
- evaluates entry and exit conditions,
- emits trade intents rather than placing orders directly.

Indicator logic is shared where it makes sense. The goal here was clarity more than cleverness.

---

### Execution and Position Tracking

Execution and position tracking are abstracted behind broker-agnostic interfaces.

This made it easier to:
- paper trade,
- simulate fills,
- and change execution behavior without touching strategy code.

Position updates are event-driven and flow back into the rest of the system.

---

### Visualization

<!-- PLACEHOLDER: UI screenshot -->
**[Figure: Live visualization interface]**

There’s a PyQtGraph-based UI that shows:
- live candlesticks and volume,
- indicator overlays,
- quote-level activity logs.

It’s mostly there so I can watch what the system is doing while it runs.  
There’s also an experimental C++/Qt visualizer I use for performance testing and layout experiments.

---

## Testing and Backtesting

The same strategy code runs for:
- historical CSV replays,
- live paper trading,
- interactive runs with the UI attached.

Backtesting usually means:
- replaying data at zero delay,
- warm-starting indicators the same way live runs do,
- and processing logs afterward to compute performance metrics.

<!-- PLACEHOLDER: Backtest artifacts -->
**[Figure: Backtest logs and analysis outputs]**

---

## Example Walkthrough

<!-- PLACEHOLDER: Demo video -->
**[Video: Live scan and paper-trade run]**

This would show:
- symbols entering and leaving the active universe,
- strategy tasks starting and stopping,
- UI updates in real time,
- and order events during a paper trading session.

---

## Limitations and Open Areas

- Data availability and rate limits depend heavily on provider.
- Python limits how far latency-sensitive ideas can go.
- Multi-feed live testing still has some rough edges.
- Risk controls are intentionally conservative while experimenting.

---

## Current Direction

Lately the work has been around:
- tightening symbol selection criteria,
- improving data ordering and warm-start behavior,
- simplifying some of the strategy state machines,
- and cleaning up how logs feed into analysis.

Next steps are likely to include:
- more standardized performance summaries,
- better experiment tracking,
- and selectively moving performance-critical pieces into C++.

