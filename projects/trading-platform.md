---
layout: project
title: "Algorithmic Trading Platform (Research & Experimentation)"
description: "A modular trading platform built for strategy design and live deployments."
year: 2025
tags: [trading, python, asyncio, systems, market-structure]
---

## Overview

This is an end-to-end algorithmic trading platform I’ve been building for myself as a platform to experiment with faster (second-scale) trading strategies. I wanted something that could run live or replay historical data through a strategy without constantly making new scripts or pipelines for everything I wanted to test.

Everything is in Python. I used pure numpy wherever possible but there's a lot of room for optimization, however, it doesn't make sense to put effort into that quite yet. I did mess around with some C++ bindings for some intensive indicator calcs, but haven't taken that very far yet.

---

## Architecture

*Scanner → Strategy Manager → Async Data Streams → Strategy State Machines → Execution (and visualization if enabled)*

At a high level, the platform is organized as a pipeline:

1. A stock selection layer produces a dynamic list of symbols.
2. A strategy manager reacts to changes in that list and spins up or shuts down data streams for each symbol.
3. Each symbol runs its own async data stream, pushing data into individual buffers for one or more strategies so they don't block each other.
4. One or more strategy state machines consume this market data.
5. Trade signals flow into an execution and position layer (although I'm still in the phase of finding a strategy I actually like enough to try, so I haven't built these out).
6. Everything is mirrored into logs and/or a live UI and/or the terminal.
7. Data streams run until out of data if backtesting, or until shut down by the manager if live (for example, at market close)

The intent was to be as modular as possible, and every layer has a dummy or testing version that can be swapped in for a more sophisticated version as needed.

---

## Core Components

### Stock Selection

Stock selection lives behind a small interface:
- `update()`
- `get_active_stock_list()`

The "scanner" depends on what I'm doing, so far the options are just:
- fixed symbol lists for testing
- premarket scanners with filters on price change, volume, float, industry, and other factors for a strategy I'm actively working on

For any new strategy, I can extend the base scanner class and plug it in.

It's tough to backtest a scanner (seeing what it would have picked on previous days), but it's probably worth the effort to build this functionality because this is a really important part of some strategies. For now, I just have it run every morning so I can build up a dataset as I go. I get a list like the one below each day, and built a pipeline to fetch the relevant data from Databento so I can then go test how the combined scanner + strategy would've done on that day. I also get a much richer log showing exactly why every candidate stock got filtered for refining the scanner.

**Example of scanner selections**  
<img src="/assets/img/scan-selections.png" alt="Scanner selections" style="max-width:70%; height:auto;">

---

### Strategy Management

The strategy manager handles lifecycle stuff:
- detecting when symbols are added or removed
- starting and stopping data feeds for each symbol
- attaching one or more strategy instances to each symbol

This structure lets a lot of symbols run concurrently. The central manager concept was also one of the first things I set up in this architecture, because I foresaw a time where if I was running multiple strategies or multiple tickers, I could have multiple entry signals at the same time. If I was trading out of one account I'd need strict control across processes to avoid trying to trade more money than I have. I haven't had to build this control yet, but the framework is in place so it won't be a huge lift.

---

### Market Data Streams

I have different implementations for live WebSocket data and CSV replay emulating a WebSocket stream for backtests, but they share a base class so the state machines themselves don't care where they're getting data from. Data streams push bars into individual buffers of every strategy connected to them, so the strategies can independently handle the bars without blocking down each other or the data feed.

Everything gets normalized into the same bar and quote schema, so again the strategies don’t care where the data came from.

---

### Strategy State Machines

Strategies are generally implemented as explicit state machines, and all have a few core handlers for new bars and quotes.

Each one:
- owns its indicators and rolling buffers for internal calcs
- evaluates entry and exit conditions
- emits trade intents rather than placing orders directly

Indicators are also implemented with a standard interface and aren't dependent on internal strategy structure, which is important both for the state machines and the visualizer.

Each strategy has an internal buffer for data. The data is tagged with a label (for example, a 1 min bar or quote) and the strategy will work through the buffer as long as it's not empty and call the relevant handlers depending on the tag.

---

### Execution and Position Tracking

All I've done so far with execution is make a dummy version that logs an entry, but it should be plug-and-play when I need to add a real execution layer. I can borrow a bit from what I built for previous Coinbase and Alpaca projects which did have actual execution layers. Most of the effort will go towards allowing the manager to control multiple positions, handle sizing, etc.

---

### Visualization

There’s a PyQtGraph-based charting utility that shows:
- live candlesticks and volume
- indicators
- a running price feed, typically NBBO quotes
- a log, which usually will include log outputs from a strategy that's running

It’s mostly a debug tool but it's also just fun to watch it run with a strategy. When doing larger scale backtests or just iterating on a strategy I can disable the visualizer, since it’s the bottleneck in terms of playback speed, often by roughly two orders of magnitude. However, this also depends on whether or not the strategy is implemented efficiently and how much it tries to use quotes vs just the much less frequent bars - in the example below the 1 minute bars have some heavy logic so the visualizer isn't actually the bottleneck.

**Visualizer Example**
<video controls preload="metadata" width="100%">
  <source src="/assets/videos/screen-cap-viz.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

---

## Strategy Focus

Most of the experimentation is around high-volatility, low-float stocks in premarket because that's where I think a retail trader would have the edge (can take higher risk and liquidity is thinner). I'm focusing on the second/minute timeframe because I would obviously never be able to compete on HFT working from a laptop in California, and anything longer might be institutional territory since they could accumulate positions over time. I've also seen some traders have consistent success in this area, so it seems like a good opportunity to try to make an algorithm that can mimic a discretionary trader to an extent.

The same system could be pointed at other asset classes or timeframes with minimal changes (everything is intentionally agnostic to the timeframe of data being supplied, and it doesn't care about start or end of day so it could work just as well on a 24/7 crypto dataset).

---

## Testing and Backtesting

Any strategy can be run individually with historical data as a test, but I also built a few larger scale backtests for strategies I'm more interested in. This means:

- replay a bunch of days of data for relevant tickers/time periods, typically defined by the datasets coming from my scanner runs each morning
- parse all the logs and collect trade performance
- do some post-processing (for example, applying slippage and fees since my current dummy execution layer doesn't model them)
- put together some stats

Example stats for a strategy that may or may not be legit (still building a bigger dataset to test "out-of-sample"):

![Stats:](/assets/img/strategy-stats.png)

Also in the interest of seeing robustness to missed entries and unexpected high-slippage events (I am trading very volatile stocks after all), I'll typically do a Monte Carlo eval on the trade history where slippage is dispersed, there's a chance of missing any given trade, and there's a chance of having an anomalous event where I pull from a much higher slippage distribution.

The slippage and missed entry percents are loosely based on experience trading by hand, but are still made up, so surviving this test doesn't automatically make a strategy valid but it's useful as a sensitivity check.

I model fills assuming I have to cross the spread on both ends.

**Monte Carlo Example**
![Scanner:](/assets/img/monte-carlo-example.png)

---

## Limitations and Open Areas

- Data availability and rate limits depend heavily on provider, and I'm trying to work with as much free data as possible so I'm fairly handicapped on actual strategy development and backtesting.
- Python limits how far latency-sensitive ideas can go, but a port to C++ wouldn't be too difficult where needed.
- Multi-ticker live testing with actual WebSocket connections from Polygon(Massive) is still not quite working as expected, I'm actively debugging this and learning about the right way to build cooperative async systems. 

---

## Current Direction

Lately the work has been around:
- building better scanners
- developing an actual strategy I want to take to live testing (Having a promising strategy is the best motivation for actually getting myself to work on the tooling)
- dev to support multiple live data streams

These are probably going to be the focus for a while, although I think I mostly have a scanner where I want it. If I can get my data feeds to a good spot and the strategy is promising, next up would be a bit of live paper trading with the integrated system, then working on the execution layer (which I actually think won't be too bad, other than controlling competing entries across multiple tickers) 


