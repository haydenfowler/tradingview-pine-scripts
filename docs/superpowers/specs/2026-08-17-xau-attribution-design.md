# XAU Attribution — Design

**Date:** 2026-08-17
**File:** `xau-attribution.pine`
**Status:** Approved

## Purpose

Decompose each XAUUSD move into three additive components — a **dollar** leg, a
**precious-complex** leg, and a **gold-idiosyncratic** residual — so that when
gold sweeps a level it is possible to tell whether the move had an external
driver or was internal to gold.

The trading use is a **veto, not a trigger**. A sweep that the dollar explains is
not a liquidity grab; it is gold repricing against USD, and fading it means
implicitly taking a dollar view with its own drivers and its own persistence. A
sweep that neither the dollar nor silver explains is gold-internal, which is what
liquidity-seeking order flow looks like.

The indicator removes a category of bad fades. It does not make the remaining
fades good — the existing read still has to carry the trade.

### Why two factors

The dollar leg alone can only say "not the dollar", which leaves *genuine
repricing* and *liquidity grab* indistinguishable. Silver separates them:

- Gold moves, silver does not → monetary / safe-haven / central-bank demand,
  specific to gold.
- Both move together → the whole precious complex repriced; something real.

A gold sweep that silver did not confirm is materially more likely to be
gold-internal order flow.

Platinum was considered and rejected: it is dominantly industrial with supply
concentrated in South Africa, so it throws idiosyncratic moves that say nothing
about gold. Real yields were considered and rejected: fundamentally the strongest
gold driver, but the accessible series are daily and lagged, so they are useless
at intraday sweep timescales.

### Why XAUxxx pairs are not used

`XAUEUR` is by definition `XAUUSD / EURUSD`. A "pair-agnostic gold value" built
from a basket of XAUxxx pairs is therefore, in log terms:

```
log(GoldBasket) = log(XAUUSD) + log(DollarBasket)
```

It is not independent information — it is the same decomposition already obtained
from XAUUSD and a dollar index, differing only by the basket weights. The
refinement worth having later is a *better dollar basket* (adding AUD and CNH,
dropping SEK), not a gold basket. That is deferred; see Out of scope.

## Behaviour

### Decomposition

Per-bar log returns are taken for gold (the chart symbol), the dollar index, and
silver. Two sequential regressions follow the Frisch–Waugh–Lovell approach: the
dollar is removed from both gold and silver, and the orthogonalised silver
residual then explains what it can of the orthogonalised gold residual.

```pine
// Step 1 — remove the dollar from gold
bD     = ta.correlation(gRet, dRet, N) * ta.stdev(gRet, N) / ta.stdev(dRet, N)
r2D    = math.pow(ta.correlation(gRet, dRet, N), 2)
gResid = gRet - bD * dRet

// Step 2 — remove the dollar from silver, then regress residual on residual
bSD    = ta.correlation(sRet, dRet, N) * ta.stdev(sRet, N) / ta.stdev(dRet, N)
sResid = sRet - bSD * dRet
bS     = ta.correlation(gResid, sResid, N) * ta.stdev(gResid, N) / ta.stdev(sResid, N)
r2S    = math.pow(ta.correlation(gResid, sResid, N), 2)
idio   = gResid - bS * sResid
```

Sequential orthogonalisation is required rather than two independent
regressions, because the dollar and silver are themselves correlated and
independent fits would double-count the shared component.

Each bar's gold return is then the sum of three parts:

```
gRet  =  (bD * dRet)  +  (bS * sResid)  +  idio
         dollar          complex           idiosyncratic
```

### The move window

The attribution shown is for the last `M` bars — the leg into the sweep — not for
a single bar. Each component is accumulated with `math.sum(..., M)`.

The three parts sum exactly to the total only if the betas were constant across
the window. They drift, so the decomposition is approximate over `M`. This is
acceptable at the intended window sizes and should not be presented as exact.

Extension of the idiosyncratic component is reported as a z-score, scaled for the
window length since it is a sum of roughly independent terms:

```pine
idioZ = math.sum(idio, M) / (ta.stdev(idio, N) * math.sqrt(M))
```

### Verdict

A component's **share** is its absolute contribution over the absolute total
move:

```pine
dollarShare = math.abs(math.sum(bD * dRet, M))   / math.abs(math.sum(gRet, M))
silverShare = math.abs(math.sum(bS * sResid, M)) / math.abs(math.sum(gRet, M))
```

A share can exceed 1 when components offset each other — for example a dollar leg
pushing gold up while the idiosyncratic leg pushes it down harder. This is not an
error; shares are displayed uncapped and are not expected to sum to 1.

When the total move is near zero the shares are undefined. Guard on a small
epsilon and return *No opinion* rather than dividing.

Verdicts are evaluated in order; the first match wins.

| Condition | Verdict |
|---|---|
| `r2D < minR2` and `r2S < minR2` | **No opinion** — correlations dead |
| total move within epsilon of zero | **No opinion** — nothing to attribute |
| `dollarShare > dollarThreshold` | **Dollar-driven** — stand down |
| `silverShare > complexThreshold` | **Complex-wide** — not a fade |
| `abs(idioZ) > idioThreshold` | **Gold-idiosyncratic** — grab candidate |
| otherwise | **Mixed** — no read |

No sign test is applied to the silver condition. The orthogonalised silver
contribution `bS * sResid` already carries its own sign through `bS`, so a share
above the threshold means silver explained that much of the move in whichever
direction it went. An additional same-sign test would be redundant.

The `minR2` gate exists so the indicator can decline to answer. A sustained
collapse in R² means gold has decoupled from both reference series, which is
itself information: in such a regime "gold-specific" is the default rather than
the exception, and the veto loses its discriminating power.

### Why the lookback is a bar count

`N` is a number of bars, not a time period. Regression stability depends on
sample size, and a time-based window would give roughly 288 bars on a 5-minute
chart and 6 bars on a 4-hour chart — the first laggy, the second unusable. A
fixed 60 bars is a reasonable "recent regime" window at every timeframe in use
(≈5 hours on 5m, ≈10 days on 4h).

## Implementation

### Data

Two security calls, both on the chart timeframe:

```pine
dxyClose = request.security(dollarSym, timeframe.period, close,
     ignore_invalid_symbol = true)
xagClose = request.security(silverSym, timeframe.period, close,
     ignore_invalid_symbol = true)
```

Default `lookahead` (off) is correct here — the values are for the current bar on
the same timeframe, so there is nothing to look ahead to.

### Display

A separate pane.

- **Histogram** of `idioZ`, coloured by verdict. Rendered grey whenever the
  verdict is *No opinion*, so a dead regime is visually obvious rather than
  something to be inferred from a number.
- **Table** in a configurable corner, showing the verdict text plus the dollar
  share, silver share, `idioZ`, and `r2D`.

`Compact mode` suppresses the histogram and reduces the table to the verdict line
alone, for use on an execution chart where the full readout is not wanted. The
full form is intended for a dedicated context layout.

This is one script with a mode switch, not two scripts.

### Drawing budget

The table is a single `table` object rebuilt on the last bar only
(`barstate.islast`), so it does not consume the per-candle line/box/label budget.
The histogram is a `plot`, which is unbounded. No pruning logic is needed.

## Inputs

| Input | Type | Default |
|---|---|---|
| Dollar symbol | symbol | `TVC:DXY` |
| Silver symbol | symbol | `OANDA:XAGUSD` |
| Regression lookback (N) | int, 20–500 | 60 |
| Move window (M) | int, 2–100 | 12 |
| Minimum R² for an opinion | float, 0–1 | 0.15 |
| Dollar-driven threshold | float, 0–2 | 0.6 |
| Complex-confirmation threshold | float, 0–2 | 0.3 |
| Idiosyncratic z threshold | float, 0–10 | 1.5 |
| Compact mode | bool | false |
| Table position | Top/Bottom × Left/Right | Top right |
| Table text size | Tiny / Small / Normal / Large | Small |

The thresholds are starting guesses. They are exposed as inputs specifically
because they are expected to be retuned against live charts.

## Edge cases

- **Warmup** — the first `N` bars have no valid regression; verdict is *No
  opinion* and the histogram is suppressed.
- **Zero variance in a reference series** — a stale feed or a holiday makes
  `ta.stdev(dRet, N)` approximately zero, which would make beta explode. Guard
  on a small epsilon and fall through to *No opinion*.
- **Missing symbol** — `ignore_invalid_symbol = true` yields `na` rather than a
  compile error; `na` returns propagate and the verdict becomes *No opinion*.
- **Session misalignment** — gold and DXY do not share bar boundaries perfectly
  intraday. Unmatched bars must produce `na` returns, never a false zero, since a
  false zero return would corrupt both the correlation and the variance.
- **Live bar** — values update until the bar closes. The verdict on an unclosed
  bar is provisional, in common with any oscillator.
- **Non-gold chart symbol** — the maths is symbol-agnostic and the script will
  run anywhere. This is not prevented, but nor is it a supported use.

## Out of scope

- Automatic sweep detection. The indicator is a continuous readout consulted when
  a sweep occurs; it does not define what a sweep is.
- Regression computed on a higher timeframe and applied to a lower-timeframe
  chart.
- A custom currency basket (AUD, CNH) in place of DXY for the dollar leg.
- Platinum, or any third factor.
- Alerts.

## Known limitations

- During a high-impact release (CPI, NFP) both legs move on the same news. The
  decomposition remains arithmetically valid but has no useful read — the move is
  neither a grab nor a fade, it is repricing.
- In low volatility both reference legs are noise and the residual is
  meaningless. The tool is informative around impulsive moves and level
  interactions, not continuously.
- The additive decomposition is approximate over the move window, as described
  above.

## Verification

There is no test runner for Pine in this repository. Verification is manual:
paste into TradingView, confirm it compiles, and check behaviour against live
charts. Two things cannot be confirmed from the repository and must be checked on
the user's account:

- that `TVC:DXY` and `OANDA:XAGUSD` both resolve under their data subscription;
- that their TradingView plan supports the multi-chart layout intended for the
  context tab.

## Documentation

Add an "XAU Attribution" section to `README.md`, following the format of the
existing indicator entries.
