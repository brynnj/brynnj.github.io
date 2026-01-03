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

For regime labeling I used the Amplitude-Based Labeler from Jonathan Shore's `tseries_patterns` library.

(As an aside, all of the content on tr8dr has been very inspiring. The possibility of starting from a good labeler to build a predictive model came from https://tr8dr.github.io/RLp1/)

This labeler identifies three states, generally uptrend downtrend and flat. The labels are forward-looking, they're not meant to be traded directly, but I'm using them as a truth label in my training data.
- `+1` for upward regimes,
- `-1` for downward regimes,
- `0` for neutral / choppy periods.

Example of labeled data:

![Example of labeled data:](/assets/img/btc_labeled.png)

As a simple assessment of the validity of the regime label, the 24 bar forward returns of the labeled dataset show significant predictive power (this is forward looking though of course)

```shell
H=24 bars
           n      mean       std  win_rate    median
label
-1      6397 -0.033619  0.177416  0.337033 -0.012761
 0     16292  0.003378  0.084168  0.524122  0.000976
 1      7844  0.021708  0.086135  0.691229  0.013007
```

There's also good persistence within the regimes, so there would be some useful, tradeable information if we could predict the labels accurately.
```shell
=== Empirical transition matrix (P[next_label | label]) ===
          to_-1    to_0    to_1
from_-1  0.9023  0.0592  0.0385
from_0   0.0234  0.9512  0.0254
from_1   0.0308  0.0531  0.9162
```
---

The goal is then to train a classifier to label the current regime as accurately as possible, without future information, so that it could be used in an actual strategy. Instead of forcing a hard decision, the model outputs a probability distribution across all three states at each timestep. These can be used as-is or combined into a single trend score metric evaluating the confidence of the label being +1 or -1 instead of 0.

---

## Model, features, and the walk-forward setup

This part is the “predictive” side of the regime experiment: take the **3-state labels** (`-1`, `0`, `+1`) from the amplitude-based labeler and train a classifier that outputs **probabilities for all three states** at each candle.

### Data alignment / joining

I start with two time-aligned inputs:

- **Labels file:** timestamps + a discrete regime label (`-1`, `0`, `+1`).  
  These labels come from the Amplitude-Based Labeler (not my code), and they’re treated as the reference definition of “trend up / chop / trend down.”
- **OHLCV (and indicators) file:** hourly candles with OHLCV, plus optional precomputed indicators (I found a kaggle dataset for this that had a bunch of indicators in there already, so I just used some of those).

### Feature creation

The feature set was a collection of stuff meant to indicatr price and volume action and things that demonstrate an active breakout. This was just my first pass - there's likely tons of room to improve this set. Some of the features include:

- 1-bar log return
- candle range/body fractions (wick/body proxies)
- ATR-ish volatility (in bps)
- Donchian-style distances to recent highs/lows (in bps)
- simple MACD histogram + slope
- volume spike / z-score style features

I used sklearn for all of the ML work - standardizing and scaling, the model itself, metrics, etc.

### Model choice

This is a **multiclass classifier** that outputs a probability vector over the three states.

I mainly used a gradient-boosted tree classifier (`HistGradientBoostingClassifier`) because it tends to behave well on mixed feature sets without a ton of tuning.

The output of each fold is:
- predicted class (`label_pred`)
- predicted probabilities (`proba_-1`, `proba_0`, `proba_1`)

Those probabilities are what later get turned into `proba_max`, the up-vs-down margin, and ultimately the trend score.

### Walk-forward process (OOS probability generation)

This runs as a **walk-forward** procedure:

- There’s an initial **warmup block** (to ensure indicators exist and to avoid training on unstable early rows).
- Then for each fold:
  - train on a contiguous historical block
  - test on the next contiguous block immediately following it
  - roll forward by a fixed step and repeat

I opted to use an **expanding training window** in most runs, meaning the training set grows over time (more like “real life” where you keep accumulating data), but there’s also a rolling-window option and it didn't seem to change performance much.

The important part: every row in `oos_regime_predictions.csv` is produced by a model that **did not train on that row** (or anything after it). That file is the out-of-sample prediction set that downstream analysis uses.

Alongside predictions, I log fold-level metrics (accuracy, balanced accuracy, log loss) mainly as sanity checks.

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

![Binned mean fwd return vs trend score](/assets/img/binned_mean_fwd_ret_vs_trend_score.png)

![Binned mean abs fwd return vs trend score](/assets/img/binned_mean_fwd_abs_ret_vs_trend_score.png)

Across horizons, the unsigned trend score has a noticeable correlation with
absolute forward returns, so higher scores do tend to indicate bigger moves. There's still little-to-no directional information in the unsigned trend score, again to be expected.

### Signed trend score (directional check)

I then tried putting a sign back in to see if the model has any meaningful predictive power directionally.

![Binned mean fwd return vs signed trend score](/assets/img/binned_mean_fwd_ret_vs_signed_trend.png)

Pretty promising with fixed-size bins (~2500 samples each) , but looking only at fixed extreme bins surprisingly adds quite a bit of noise instead of a cleaner signal:

![Binned mean fwd return vs signed trend score (fixed bins)](/assets/img/binned_mean_fwd_ret_vs_signed_trend_fixed_bins.png)


Overall, this regime prediction method seems more effective as a volatility signal than a directional indicator. Directional signal seems like it might exist, but it's pretty weak and sensitive to how it's sliced.

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
- Options framing: potentially useful in an options strat depending on how predicted move sizes compare to implied volatility. It's possible that everything this model predicts would already be priced in.

---

## Next Steps

I really barely scratched the surface of the predictive model - this was my first try at feature set, model selection, training approach, etc. and I might be able to do significantly better if I work more at this (especially including features that aren't just price and volume action, news and sentiment would probably go a long way). I first want to see if I can find an application for this in a metalabeler or otherwise, and if so I may revisit this to see how far I can push it.
