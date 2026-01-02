---
layout: project
title: "Regime Labeling"
description: "Exploring regime labels, conviction scores, and their relationship to forward returns."
---

# Regime Labeling Experiments

I've been thinking about regime labeling as a filter on trade entries for a
long time. Everything seems to eventually assume trending mean-reverting. When
strategies struggle, it often feels like the removing bad entries when in the wrong market is all that's needed.

This part of the project is an attempt to explore that idea more directly.
Instead of trying to trade regimes outright, the goal here was to see whether
regime structure can be defined in a reasonable way, and whether a model trained
against that structure produces signals that line up with anything observable
in future returns.

---

## Regime labels as a reference

For regime definitions, I didn't invent a custom labeling scheme. I used the
Amplitude-Based Labeler from Jonathan Shore's `tseries_patterns` library.

(As an aside, all of the content at tr8dr has been very inspiring. The possibility of starting from a good labeler to build a predictive model came from https://tr8dr.github.io/RLp1/)

This labeler segments price into three broad states based on realized movement:
- sustained upward moves,
- sustained downward moves,
- and periods of chop / no meaningful direction.

![Example of labeled data:](/assets/img/btc_labeled.png)

The forward (ex. 24 bar) returns of the labeled dataset show significant predictive power (this is forward looking though of course)

```shell
H=24 bars
           n      mean       std  win_rate    median
label
-1      6397 -0.033619  0.177416  0.337033 -0.012761
 0     16292  0.003378  0.084168  0.524122  0.000976
 1      7844  0.021708  0.086135  0.691229  0.013007
```

A few caveats up front:
- The labels are forward-looking and non-causal.
- They're not meant to be traded directly.
- I'm using them as a truth label in my training data.

---

## A three-state regime classifier

The regime model trained on these labels is explicitly three-state, not binary:
- `+1` for upward regimes,
- `-1` for downward regimes,
- `0` for neutral / choppy periods.

Instead of forcing a hard decision, the model outputs a probability distribution
across all three states at each timestep. That distribution turns out to be more
informative than the final predicted label on its own.

In particular, it lets you separate a few different questions:
- Is the model confident at all?
- Is it confident that the market is directional rather than choppy?
- If directional, does it lean up or down?

The trend score comes from trying to summarize those ideas into a single
continuous value.

---

## Model, features, and the walk-forward setup

This part is the “predictive” side of the regime experiment: take the **3-state labels** (`-1`, `0`, `+1`) from the amplitude-based labeler and train a classifier that outputs **probabilities for all three states** at each hour.

### Data alignment / joining

I start with two time-aligned inputs:

- **Labels file:** timestamps + a discrete regime label (`-1`, `0`, `+1`).  
  These labels come from the Amplitude-Based Labeler (not my code), and they’re treated as the reference definition of “trend up / chop / trend down.”
- **OHLCV (and indicators) file:** hourly candles with OHLCV, plus optional precomputed indicators.

The key step is a **timestamp join** so the model only trains on rows where the OHLCV row and the label row refer to the same hour. If the join produces very few rows, that’s usually a timestamp cadence / timezone mismatch issue.

### Feature creation

Feature-wise, I tried to keep it simple and interpretable. The feature set is:

1) **Core OHLCV numeric columns**  
Open, high, low, close, volume.

2) **Precomputed indicator columns (recommended)**  
If the OHLCV file already includes numeric indicator columns (RSI, moving averages, BBands, etc.), I include them directly rather than re-deriving everything. In practice, this mattered a lot for performance.

3) **A small set of extra, causal OHLCV-derived features**  
These are computed using rolling windows and shifts so they only use information available up to that point (no lookahead). The ones I kept were mostly “robust shape” and “participation” signals:
- 1-bar log return
- candle range/body fractions (wick/body proxies)
- ATR-ish volatility (in bps)
- Donchian-style distances to recent highs/lows (in bps)
- simple MACD histogram + slope
- volume spike / z-score style features

After feature computation, rows with NaNs (mostly caused by rolling indicators needing warmup) get dropped so the model sees a clean matrix.

### Model choice

This is a **multiclass classifier** that outputs a probability vector over the three states.

I mainly used a gradient-boosted tree classifier (`HistGradientBoostingClassifier`) because it tends to behave well on mixed feature sets without a ton of tuning. I also had a multinomial logistic regression option as a baseline (with standardization), but the boosted model was the primary one.

The output of each fold is:
- predicted class (`label_pred`)
- predicted probabilities (`proba_-1`, `proba_0`, `proba_1`)

Those probabilities are what later get turned into `proba_max`, the up-vs-down margin, and ultimately the trend score.

### Walk-forward process (OOS probability generation)

I didn’t want a single static train/test split because regimes drift, and because it’s too easy to accidentally benefit from future distribution information. So this runs as a **walk-forward** procedure:

- There’s an initial **warmup block** (to ensure indicators exist and to avoid training on unstable early rows).
- Then for each fold:
  - train on a contiguous historical block
  - test on the next contiguous block immediately following it
  - roll forward by a fixed step and repeat

I used an **expanding training window** in most runs, meaning the training set grows over time (more like “real life” where you keep accumulating data), but there’s also a rolling-window option.

The important part: every row in `oos_regime_predictions.csv` is produced by a model that **did not train on that row** (or anything after it). That file is basically the “clean” out-of-sample probability stream that downstream analysis uses.

Alongside predictions, I log fold-level metrics (accuracy, balanced accuracy, log loss) mainly as sanity checks. I don’t treat those as the main success metric — the more relevant question for this project is whether the probability stream produces anything useful when conditioned against forward returns.

## Trend score: what it's trying to measure

Rather than treating regime prediction as a classification problem, I focused
on extracting a confidence-weighted signal from the model's probabilities.

The trend score combines two pieces:

1. Overall confidence  
   The highest probability among the three states. This captures how decisive
   the model is, regardless of direction.

2. Directional separation  
   The difference between the probabilities of the up and down states. This
   ignores the neutral class and focuses purely on how asymmetric the
   directional beliefs are.

Multiplying these together produces an unsigned trend score that's large when:
- the model is confident, and
- that confidence is concentrated in one directional side rather than split or
  dominated by neutrality.

A signed version simply applies the sign of the up-minus-down probability
difference, which allows for directional tests without changing the underlying
magnitude.

At this stage, the trend score isn't intended to be a trading signal. It's just
a way to collapse a three-state probability vector into something easier to
analyze.

---

## Diagnostics and relationships

Rather than optimizing classification accuracy, I looked at how the trend score
relates to forward returns, across multiple horizons.

### Trend score vs forward return

The first pass is a raw scatter: do higher scores line up with anything
interesting later? Unsurprisingly they don't, because it's unsigned.

![Trend score vs fwd return](/assets/img/scatter_trend_score_vs_fwd_ret_6.png)

### Binned mean forward return

Binning the score and looking at average forward returns makes broader patterns
easier to see.

![Binned mean fwd return vs trend score](/assets/img/binned_mean_fwd_ret_vs_trend_score.png)

![Binned mean abs fwd return vs trend score](/assets/img/binned_mean_fwd_abs_ret_vs_trend_score.png)

Across horizons, the unsigned trend score lines up much more clearly with
absolute forward returns than with signed returns. In other words, higher scores
do tend to indicate bigger moves.

### Signed trend score (directional check)

I then tried putting a sign back in to see if the model has any meaningful predictive power directionally.

![Binned mean fwd return vs signed trend score](/assets/img/binned_mean_fwd_ret_vs_signed_trend.png)

Pretty promising with fixed-size bins (~2500 samples each) , but looking only at fixed extreme bins surprisingly adds quite a bit of noise instead of a cleaner signal:

![Binned mean fwd return vs signed trend score (fixed bins)](/assets/img/binned_mean_fwd_ret_vs_signed_trend_fixed_bins.png)

---

A few patterns show up fairly consistently, though I'd still treat these as
observations rather than conclusions:

- It behaves more like a volatility signal than a directional indicator.
- Directional signal seems like it might exist, but it's pretty weak and sensitive to how it's
  sliced.

That suggests this layer is probably more useful as context or a feature in a broader model than as a primary
entry driver.

---

## How this might be used

Some possible uses that seem consistent with the behavior above:

- Trade filtering: avoid momentum-dependent strategies when conviction is low.
- Sizing and risk: scale size, stops, or targets based on expected move
  magnitude.
- Strategy gating: allow certain strategies to operate only when the regime
  score clears a threshold. Given the noise here, I would probably prefer this as part of a metalabeling system rather than a standalone decision gate.
- Options framing: potentially useful in an options strat depending on how realized movement compares to implied volatility.

---

## Next Steps

I really barely scratched the surface of the predictive model - this was my first try at feature set, model selection, training approach, etc. and I might be able to get significantly better labels if I work more at this (especially including features that aren't just price and volume action, news and sentiment would probably go a long way). I first want to see if I can find an application for this in a metalabeler or otherwise, and if so I may revisit this to see how far I can push it.
