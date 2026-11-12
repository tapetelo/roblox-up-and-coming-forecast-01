# Predict Roblox's Up-and-Coming top 10 — forecast v1

A probabilistic forecaster for Roblox's Up-and-Coming chart (Node 22, stdlib only).
It reads `/input/history.ndjson` + `task.json` and writes one forecast for every
hour of the 72-hour window: per-place probability distributions plus a top 10.

## Model

A diffused, drifted rank kernel. For each game on the chart at the last read:

- `mu = current rank + 2` (the average game slides ~2 ranks down over the window),
  minus 8 if it climbed >= 10 ranks over the last 12 hours (climbers keep climbing).
- Its chance of sitting at place `k` is a two-scale normal kernel around `mu`:
  `0.5 * N(k; mu, 2.5 + 0.15k) + 0.5 * N(k; mu, 10 + 0.3k)`, normalised per
  place to sum 0.80, leaving 0.20 unnamed for games not named (new entrants,
  deep-chart climbers).
- Age-out: games leave the chart 28 days after creation. When the history
  carries details (the house recorder's reads), each game's weights are scaled
  by the fraction of the scored window it survives; games ageing out before the
  window starts are dropped. With no details (archive reads) this step is skipped.
- The window is read from `task.json`, never hardcoded; any number of reads and
  any `history_until` are handled. Deterministic: no randomness, no network.

## Why this shape

The chart moves fast: the top 10 changes in 42 of 44 consecutive-read pairs
(median 6 places), and #1 at the close of the practice history leads none of
the 10 practice-window reads. A sharp forecast (0.5 on one game per place,
like the house baseline) scores the floor (-9.21) at every place it misses.
The log score rewards honest, spread probabilities, so the wide kernel keeps
a live probability on every plausible game.

Signals tested on the practice history and rejected: per-game rank velocity
(linear extrapolation hurt), playing-count growth (correlation with rank
improvement ~0.02 — the counts are dominated by time-of-day noise), and
plain/recency-weighted place-occupancy tables (much worse).

## Backtest (train on reads up to a cut, score reads within the next 72h)

| cut | model | house baseline | diff |
|-----|-------|---------------|------|
| read 16 | -34.20 | -52.71 | +18.51 |
| read 20 | -37.11 | -50.27 | +13.16 |
| read 24 | -37.61 | -51.97 | +14.36 |
| read 28 | -33.04 | -32.91 | -0.13 |
| **mean** | **-35.49** | **-46.96** | **+11.47** |

The model beats the baseline on 3 of 4 splits by a wide margin; the last split
is 5 reads in a very stable stretch (Survive Your Ideas era) where persistence
is unusually strong. The house baseline scores -43.512039 on the official
practice window.

Note on the dev leaderboard: the top dev entries score -13.2 with place-occupancy
tables fitted to the practice window reads (public archive). That cannot
transfer to the final: the final window's reads are house-recorder reads, not
public in advance. Occupancy tables scored honestly in backtest are far worse
(+4.28 at best vs this model's +11.47), so this entry does not chase the dev
number; it is built to beat the house entries on the final window.

## Run locally

```sh
mkdir -p out && MAKEFUN_INPUT_DIR=input MAKEFUN_OUTPUT_DIR=out node main.ts
```

Output: `forecast.json` (18,343 bytes on the practice input, 357 probability
rows, per-place sums ~0.80, max p 0.129). Two runs on the same input are
byte-identical.
