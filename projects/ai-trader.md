---
title: "AI Trader: LLM-Driven ETH Perps"
layout: project
tags: [trading, llm, crypto, python, automation]
---

## Overview

This is an experimental crypto trading system that uses an LLM to generate trade decisions for ETH perpetual futures.

The core idea was to see what happens if you give an LLM a compact snapshot of market context and ask it to make a concrete trading decision — entry, direction, sizing, and exits — under strict constraints.

I do want to clarify, I wasn't expecting this to be an actual scientifically sound project. It was more for fun, to get familiar with the relevant APIs, and building a system around LLM decision making with built-in guardrails.

Some crypto seems to sit in an interesting middle ground:
- liquid enough that slippage is manageable,
- volatile enough that there’s money to be made,
- some evidence here and there that there are potential inefficiencies still (example: https://www.mdpi.com/2227-7390/10/13/2338)

I ended up using ethereum perpetual futures for this project because they had just launched on Coinbase and had a promo with extremely low fees.

One constraint was that I built this while working around free ChatGPT API usage limits, which meant prompts had to stay concise and decision frequency had to be limited. Running on a 30 min interval was about as frequent as I could get while still keeping enough context and instructions in the prompt.

---

## Structure

**System Flow**
![Flow diagram](/assets/img/ai_trader_flow.png)

The system is split into a few stages:

1. **Data collection** builds a snapshot of recent market state.
2. **LLM decisioning** produces a structured trade decision, with a brief description of the logic behind the decision so I can go back and try to understand why it took good and bad trades.
3. **Safety validation** checks the decision against hard constraints to prevent runaway sizing.
4. **Execution** places or manages orders via a broker abstraction. Initially this was just a placeholder that tracked entry and exit prices to paper trade, and later I swapped it out for an actual Coinbase interface.

A scheduler runs this on 30 min intervals (just used cron on an EC2 instance). The first step is to check if there's already an open position, and if so the cycle is skipped before the data collection step starts.

The LLM is intentionally isolated behind schemas and validation logic. It never talks to an exchange directly.

---

## Data & Prompting

Each decision cycle builds a snapshot that includes:
- recent price and volume bars (30m),
- higher-timeframe context (daily indicators),
- a short summary of recent relevant news (including adjacent news like BTC and general market sentiment, to capture the risk-on or risk-off feel).

Prompting included context about what it was doing and its objective, some guidance on how to interpret the provided data, and some nudges to bias it towards momentum. 

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
- no new entries if a position is already open

If anything fails validation, the cycle is skipped.

---

## Execution

Execution runs through a broker abstraction so the same logic supports paper trading and live exchange-backed trading.

Order placement, position tracking, and exits are handled outside the LLM loop.

All of this was deployed on an AWS EC2 instance that I spun up for myself a year ago. I have a framework reasonably well abstracted for strategies like this that run at scheduled intervals, so this shared a good chunk of code with the min variance deployment on Alpaca.

(as an aside, I also built a super simple dashboard for these strategies, so the EC2 instance also has code for that backend, but I haven't really taken that anywhere yet)

---

## Results

Behavior depended heavily on market regime and prompt bias.

Early on, I paper traded the system during a time where ETH was making big moves (May through September 2025). The prompts were momentum-oriented so performance looked very good.

Since I switched to live trading, ETH has been coming back down. There were still a few moves in October that lasted long enough for a momentum strategy to do well, but the price drop since and many failed breakouts mean this strategy keeps bleeding money, and has given back pretty much all of the initial gains.

**Trade timeline and equity curve for real-money testing**
![Equity curve](/assets/img/ai_trader_equity_curve.png)

Over the full tested period, it did outperform holding ETH outright and stayed out of major downtrends. I would like to leave it on until ETH sees a sizeable move upwards again and I get to see it performing in the regime it was built for, but I can't really wait out the drawdown, so for now I've disabled it. The focus needs to be on a way to backtest this if I want to try any more with it.

---

## Takeaways So Far

In general, it doesn't do as well as I'd hope at bringing in broader market trends and sentiment. I think this is the result of bad features more than anything, but I'd like it to have a better understanding of risk-on vs risk-off sentiment and tie that to it's assessment of whether or not a breakout is likely to continue.

One thing I haven’t implemented yet but would like to try is feeding recent trade performance back into the snapshot in a very compact form. Even minimal self-feedback might help the system adjust across regimes. Also, some notion of sentiment (going beyond just news) could go a long way - when I started this I was using cryptopanic and had a significantly richer feature set, but they upped prices significantly and so I moved away from that.

Unsurprisingly, not being able to backtest is a huge handicap, every change is a guess and I only get to see performance in whatever market exists now.

---

## Next Steps

The most important next step would be to figure out a way to backtest without blowing through tokens. Not sure yet how to pull that off other than a) eat the cost or b) get a strong enough machine (still eating the cost) to run the decisions locally with a powerful model off ollama.

Longer term, this project is probably heading into more traditional model-based or RL-driven approaches using a similar feature set. The LLM doesn't appear to be adding all that much in the way of interpreting price and volume action, so it's just becoming a feature engineering exercise, and it would help to get around all the token use constraints holding me back. I have a metalabeling project in the works.

For now, it’s a useful sandbox for building a system around LLM reasoning, safeguarding it well enough to actually deploy my own money live, and building a robust live pipeline that ran and generated results for months without issues or intervention.
