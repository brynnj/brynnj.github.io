---
title: "Strategy Lab: LLM-Driven Strategy Evolver"
description: "A local, LLM-guided loop that proposes unconventional trading strategies, executes them in a sandboxed runner, and scores them with quick portfolio metrics."
date: 2026-01-01
layout: project
tags: [docker, llm, trading, backtesting, python, ollama]
---

# Strategy Lab: LLM-Driven Strategy Evolver

## Overview
Strategy Lab is a lightweight experimentation loop that asks a local LLM (via Ollama) to propose novel, non-indicator-based trading ideas, runs the generated `run_strategy()` code, and scores performance with simple Sharpe, drawdown, and total return metrics. It logs prompts, code, and metrics for quick iteration and post-hoc analysis.

## Problem
I wanted a fast way to explore unconventional strategy ideas without hand-coding each one, while still keeping a reproducible evaluation loop and basic metrics to rank or reject candidates. This came to mind when I first learned about ollama/the ability to run an LLM locally in general. I realized after starting that this is really a poor man's agent workflow and could be done way better if I just pay a bit for some real tools, but it was a useful exercise in learning how to build such a system.

## Approach / Architecture
A single-iteration loop prompts an LLM for a concise strategy and a `run_strategy()` function. The code block is parsed, written to a temporary module, dynamically imported, and executed to produce a DataFrame of returns. Metrics are computed and written to a log file so the prompt/response/score triad stays traceable. Everything is containerized and the sandbox is given very limited permissions in case the LLM produces any malicious code.

## Key technical details
- `strategy_lab/main.py`: orchestrates the prompt -> model generate -> execute -> score pipeline and logs each run to `data/`.
- `strategy_lab/models/ollama_model.py`: streams tokens from a local Ollama server (uses `host.docker.internal` when inside Docker).
- `strategy_lab/utils/code_parser.py`: extracts fenced Python code blocks for execution.
- `strategy_lab/utils/strategy_runner.py`: writes code to a temp file, dynamically imports it, executes `run_strategy()`, then cleans up.
- `strategy_lab/utils/metrics.py`: computes Sharpe, max drawdown, and total return from cumulative returns.
- `strategy_lab/run_dev.py` + `strategy_lab/dev_strategy.py`: manual strategy testing workflow for debugging outside the LLM loop.

## Results
- My laptops are nothing special and the best I could do locally was deepseek-coder:6.7b. This is far too weak at reasoning to actually produce interesting strategies and iterate on results, and even the coding was so weak that most attempts had a syntax error or some other issue preventing me from even getting outputs. I think this has a lot of promise for both trading and many other projects, but I need to either invest in a good machine or start paying for API credits for a frontier model.

## Limitations
- Executes model-generated code directly; no sandboxing beyond temp-file cleanup.
- Metrics are minimal and assume daily data with annualization constant fixed at 252.
- Lacks dataset/version tracking and experiment orchestration (no seeds or reproducible configs).
- No automated tests or validation for malformed strategies or missing columns.

## Next steps
- Add strict validation around `run_strategy()` outputs and a safer execution sandbox.
- Log input data provenance and strategy metadata (ticker, date range, parameters).
- Expand metrics (e.g., Calmar, turnover, exposure) and add baseline comparisons.
- Add a small experiment harness (e.g., multi-run sweep + leaderboard).
