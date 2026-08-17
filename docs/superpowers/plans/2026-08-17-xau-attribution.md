# XAU Attribution — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `xau-attribution.pine`, an indicator that decomposes each XAUUSD move into a dollar leg, a precious-complex leg, and a gold-idiosyncratic residual, and reports a plain-English verdict on which one drove the recent move.

**Architecture:** A single new `.pine` file, non-overlay (separate pane). Two `request.security` calls fetch a dollar index and silver on the chart timeframe. Log returns feed two sequential regressions (Frisch–Waugh–Lovell): the dollar is regressed out of both gold and silver, then the orthogonalised silver residual explains what it can of the orthogonalised gold residual. What remains is idiosyncratic. The three components are accumulated over a short move window and compared to produce a verdict, which drives both a histogram colour and a corner table.

**Tech Stack:** Pine Script v6, TradingView indicator. No test framework — verification is done by loading the script in the TradingView Pine Editor and inspecting behaviour on a live chart.

## Global Constraints

- Pine Script v6 (`//@version=6`).
- File begins with the MPL 2.0 comment line used by the other scripts in this repo.
- Indicator title `"XAU Attribution"`, shorttitle `"XAU ATTR"`, `overlay=false`.
- All regression maths uses **log returns**, never raw price differences.
- `EPS = 1e-10` is the shared guard against division by a near-zero standard deviation.
- A reference series that did not print a new bar is **stale**; its return must be
  `na`, never `0`. Detected by comparing the reference bar's `time` to its previous
  value — see Task 2.
- Variance and correlation inputs must never have `na` replaced by `0`
  (it would deflate the standard deviation and inflate R²). Sliding **sums** over the
  move window may use `nz()`, because a bar with no new information contributes no
  movement — this asymmetry is deliberate.
- No `line`, `box`, or `label` objects are used, so no drawing-budget pruning is needed.

## Verification method

There is no automated test runner for Pine Script. Every task that changes
`xau-attribution.pine` is verified the same way:

1. Open <https://www.tradingview.com/> and open the Pine Editor.
2. Paste the full contents of `xau-attribution.pine` into the editor.
3. Click **Add to chart**.
4. Confirm there is no red compilation error in the editor's console.
5. Inspect the pane against the task's stated expectation.

Use **`OANDA:XAUUSD` on a 15-minute chart** for inspection. That is the middle of
the intended range and busy enough that the reference series update on most bars.

**Before Task 2, confirm both reference symbols resolve on this account.** In the
Pine Editor, open a new chart tab and type `TVC:DXY` then `OANDA:XAGUSD` into the
symbol search. If either does not resolve, substitute a working equivalent
(`ICEUS:DX1!` or `CAPITALCOM:DXY` for the dollar; `TVC:SILVER` or the broker's own
`XAGUSD` for silver) and use that as the default throughout the remaining tasks.

---

### Task 1: Create the script skeleton, inputs and style mapping

**Files:**
- Create: `xau-attribution.pine`

**Interfaces:**
- Consumes: nothing.
- Produces: input variables `dollarSym`, `silverSym` (string); `lenReg`,
  `lenMove` (int); `minR2`, `dollarThreshold`, `complexThreshold`,
  `idioThreshold` (float); `compactMode` (bool); `tablePosInput`,
  `tableSizeInput` (string); and the derived constants `tablePos`, `tableSize`,
  `EPS`.

- [ ] **Step 1: Create the file with the header, declaration, inputs and style mapping**

Create `xau-attribution.pine` with exactly this content:

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/

//@version=6
indicator("XAU Attribution", shorttitle="XAU ATTR", overlay=false)

// USER SETTINGS

dollarSym = input.symbol("TVC:DXY", title="Dollar Symbol", group="Reference Series",
     tooltip="Dollar index used as the first regression factor. ICEUS:DX1! is an alternative if your plan carries it.")
silverSym = input.symbol("OANDA:XAGUSD", title="Silver Symbol", group="Reference Series",
     tooltip="Silver, used to detect precious-complex-wide moves after the dollar has been removed.")

lenReg  = input.int(60, title="Regression Lookback (bars)", minval=20, maxval=500, group="Calculation",
     tooltip="Sample size for the rolling regressions. A bar count, not a time period — regression stability depends on the number of observations.")
lenMove = input.int(12, title="Move Window (bars)", minval=2, maxval=100, group="Calculation",
     tooltip="How many recent bars make up 'the move' being attributed — the leg into the sweep.")

minR2            = input.float(0.15, title="Minimum R² for an Opinion", minval=0.0, maxval=1.0, step=0.05, group="Thresholds",
     tooltip="Below this, both relationships are treated as dead and the verdict becomes NO OPINION.")
dollarThreshold  = input.float(0.6, title="Dollar-Driven Threshold", minval=0.0, maxval=2.0, step=0.05, group="Thresholds")
complexThreshold = input.float(0.3, title="Complex-Confirmation Threshold", minval=0.0, maxval=2.0, step=0.05, group="Thresholds")
idioThreshold    = input.float(1.5, title="Idiosyncratic Z Threshold", minval=0.0, maxval=10.0, step=0.1, group="Thresholds")

compactMode    = input.bool(false, title="Compact Mode", group="Display",
     tooltip="Hides the histogram and all table rows except the verdict, for use on an execution chart.")
tablePosInput  = input.string("Top Right", title="Table Position", options=["Top Left", "Top Right", "Bottom Left", "Bottom Right"], group="Display")
tableSizeInput = input.string("Small", title="Table Text Size", options=["Tiny", "Small", "Normal", "Large"], group="Display")

// CONSTANTS

EPS = 1e-10

// STYLE MAPPING

tablePos = switch tablePosInput
    "Top Left"     => position.top_left
    "Top Right"    => position.top_right
    "Bottom Left"  => position.bottom_left
    "Bottom Right" => position.bottom_right
    => position.top_right

tableSize = switch tableSizeInput
    "Tiny"   => size.tiny
    "Small"  => size.small
    "Normal" => size.normal
    "Large"  => size.large
    => size.small
```

- [ ] **Step 2: Verify it compiles and the settings dialog is correct**

Follow the **Verification method** above.

Expected: no compilation error. The indicator adds an empty pane below the chart.
Opening its settings shows four groups in order — "Reference Series" (Dollar
Symbol `TVC:DXY`, Silver Symbol `OANDA:XAGUSD`), "Calculation" (Regression
Lookback 60, Move Window 12), "Thresholds" (four floats), and "Display" (Compact
Mode unticked, Table Position "Top Right", Table Text Size "Small").

- [ ] **Step 3: Commit**

```bash
git add xau-attribution.pine
git commit -m "feat: add XAU attribution indicator skeleton and inputs"
```

---

### Task 2: Fetch reference series with staleness detection, and compute log returns

**Files:**
- Modify: `xau-attribution.pine` (append after the STYLE MAPPING block)

**Interfaces:**
- Consumes: `dollarSym`, `silverSym` from Task 1.
- Produces: `gRet`, `dRet`, `sRet` (series float log returns, `na` when the
  source bar is missing or stale).

- [ ] **Step 1: Append the data and returns blocks**

Append the following to the end of `xau-attribution.pine`:

```pine

// DATA
// The bar's `time` is fetched alongside `close` because request.security
// forward-fills: when the reference symbol has no bar at this timestamp it
// returns the PREVIOUS close, which would compute as a return of exactly 0.
// A non-advancing bar time is how that stale repeat is detected.

[dxyClose, dxyTime] = request.security(dollarSym, timeframe.period, [close, time], ignore_invalid_symbol = true)
[xagClose, xagTime] = request.security(silverSym, timeframe.period, [close, time], ignore_invalid_symbol = true)

isFresh(t) =>
    na(t) ? false : na(t[1]) ? false : t != t[1]

// RETURNS
// na, never 0, when the bar is missing or stale — a false zero would deflate the
// standard deviation and inflate the correlation.

logRet(src) =>
    prev = src[1]
    na(src) or na(prev) or src <= 0 or prev <= 0 ? na : math.log(src / prev)

gRet = logRet(close)
dRet = isFresh(dxyTime) ? logRet(dxyClose) : na
sRet = isFresh(xagTime) ? logRet(xagClose) : na

// TEMPORARY — removed in Task 3
plot(gRet, title="Gold return", color=color.yellow)
plot(dRet, title="Dollar return", color=color.orange)
plot(sRet, title="Silver return", color=color.silver)
```

- [ ] **Step 2: Verify the reference series resolve and the returns are live**

Follow the **Verification method** above on `OANDA:XAUUSD`, 15-minute.

Expected:

- No compilation error, and no "invalid symbol" message in the pane.
- Three lines oscillating around zero, magnitudes on the order of 0.001 to 0.01.
- Hovering a bar shows all three values in the Data Window, and the Dollar return
  is generally the *opposite* sign to the Gold return.

- [ ] **Step 3: Verify staleness detection is not suppressing most bars**

In the Data Window, scroll back over roughly 100 bars of recent regular trading
hours and count how often "Dollar return" reads `n/a`.

Expected: `n/a` on a small minority of bars, concentrated around the daily
rollover and the weekend gap.

**If more than about a third of active-session bars are `n/a`,** the chosen
dollar symbol is on a materially different bar grid to gold. Stop and switch the
Dollar Symbol input to an alternative (`ICEUS:DX1!` or `CAPITALCOM:DXY`), re-check,
and change the default in Step 1's code to whichever works. This is the single
most likely cause of the finished indicator sitting permanently on NO OPINION, so
it is worth settling now rather than discovering it at Task 5.

- [ ] **Step 4: Commit**

```bash
git add xau-attribution.pine
git commit -m "feat: fetch dollar and silver series with staleness detection"
```

---

### Task 3: Regress out the dollar

**Files:**
- Modify: `xau-attribution.pine` — replace the TEMPORARY plots from Task 2

**Interfaces:**
- Consumes: `gRet`, `dRet` from Task 2; `EPS`, `lenReg` from Task 1.
- Produces: `sdG`, `sdD` (series float), `bD` (series float, gold's beta to the
  dollar), `r2D` (series float, 0–1), `gResid` (series float, gold return with
  the dollar removed).

- [ ] **Step 1: Replace the temporary plots with the dollar regression**

Find this block at the end of the file:

```pine
// TEMPORARY — removed in Task 3
plot(gRet, title="Gold return", color=color.yellow)
plot(dRet, title="Dollar return", color=color.orange)
plot(sRet, title="Silver return", color=color.silver)
```

Replace it with:

```pine
// REGRESSION STEP 1 — remove the dollar from gold
// beta = correlation * (sd of y / sd of x) is the closed-form OLS slope, which
// avoids a manual loop over the window.

sdG    = ta.stdev(gRet, lenReg)
sdD    = ta.stdev(dRet, lenReg)
corrGD = ta.correlation(gRet, dRet, lenReg)

bD  = na(corrGD) or na(sdG) or na(sdD) or sdD < EPS ? na : corrGD * sdG / sdD
r2D = na(corrGD) ? na : corrGD * corrGD

gResid = na(bD) or na(gRet) or na(dRet) ? na : gRet - bD * dRet

// TEMPORARY — removed in Task 4
plot(r2D, title="R2 dollar", color=color.orange)
plot(bD, title="Beta dollar", color=color.red)
```

- [ ] **Step 2: Verify the regression produces sane values**

Follow the **Verification method** above on `OANDA:XAUUSD`, 15-minute.

Expected:

- "R2 dollar" stays between 0 and 1 at all times, never negative, never above 1.
  It should spend most of its time somewhere between 0.05 and 0.6 and visibly
  move around — a flat line pinned at 0 or 1 means something is wrong.
- "Beta dollar" is **negative** most of the time. This is the core sanity check:
  gold and the dollar are inversely related, so gold's beta to the dollar index
  should have a negative sign. A persistently positive beta means the dollar
  symbol is inverted or wrong.
- The first 60 bars of chart history show `n/a` for both, as the window fills.

- [ ] **Step 3: Verify the lookback input takes effect**

Change "Regression Lookback (bars)" to `200`.

Expected: both series become visibly smoother, and the `n/a` warmup region at the
left edge of the chart grows to 200 bars. Reset to `60`.

- [ ] **Step 4: Commit**

```bash
git add xau-attribution.pine
git commit -m "feat: regress the dollar leg out of gold returns"
```

---

### Task 4: Orthogonalise silver and isolate the idiosyncratic component

**Files:**
- Modify: `xau-attribution.pine` — replace the TEMPORARY plots from Task 3

**Interfaces:**
- Consumes: `sRet`, `dRet` from Task 2; `gResid`, `sdD`, `sdG` from Task 3;
  `EPS`, `lenReg`, `lenMove` from Task 1.
- Produces: `sResid` (series float, silver with the dollar removed), `bS`
  (series float), `r2S` (series float, 0–1), `idio` (series float, the
  gold-idiosyncratic per-bar return), `idioZ` (series float, the move window's
  idiosyncratic component as a z-score).

- [ ] **Step 1: Replace the temporary plots with the silver step**

Find this block at the end of the file:

```pine
// TEMPORARY — removed in Task 4
plot(r2D, title="R2 dollar", color=color.orange)
plot(bD, title="Beta dollar", color=color.red)
```

Replace it with:

```pine
// REGRESSION STEP 2 — remove the dollar from silver, then regress residual on residual
// Sequential orthogonalisation (Frisch–Waugh–Lovell). Two independent
// regressions would be wrong here: the dollar and silver are themselves
// correlated, so each would claim the shared component and the parts would
// over-explain the whole.

sdS    = ta.stdev(sRet, lenReg)
corrSD = ta.correlation(sRet, dRet, lenReg)

bSD    = na(corrSD) or na(sdS) or na(sdD) or sdD < EPS ? na : corrSD * sdS / sdD
sResid = na(bSD) or na(sRet) or na(dRet) ? na : sRet - bSD * dRet

sdGr    = ta.stdev(gResid, lenReg)
sdSr    = ta.stdev(sResid, lenReg)
corrRes = ta.correlation(gResid, sResid, lenReg)

bS  = na(corrRes) or na(sdGr) or na(sdSr) or sdSr < EPS ? na : corrRes * sdGr / sdSr
r2S = na(corrRes) ? na : corrRes * corrRes

idio = na(bS) or na(gResid) or na(sResid) ? na : gResid - bS * sResid

// EXTENSION
// The move window sums roughly independent per-bar terms, so its standard
// deviation scales with the square root of the window length.

sdIdio = ta.stdev(idio, lenReg)
moveIdio = math.sum(nz(idio), lenMove)
idioZ = na(sdIdio) or sdIdio < EPS ? na : moveIdio / (sdIdio * math.sqrt(lenMove))

// TEMPORARY — removed in Task 5
plot(idioZ, title="Idio Z", style=plot.style_histogram, color=color.fuchsia, linewidth=3)
plot(r2S, title="R2 silver", color=color.blue)
```

- [ ] **Step 2: Verify the idiosyncratic component and its z-score**

Follow the **Verification method** above on `OANDA:XAUUSD`, 15-minute.

Expected:

- "R2 silver" stays between 0 and 1, and is typically *lower* than "R2 dollar"
  was in Task 3 — it is explaining only what the dollar left behind.
- "Idio Z" oscillates around zero as a histogram, mostly within roughly ±2, with
  occasional excursions beyond ±3. A z-score that sits permanently above 5, or
  that never leaves ±0.2, means the `math.sqrt(lenMove)` scaling is wrong.
- The warmup region is `n/a` as before.

- [ ] **Step 3: Verify the move window input takes effect**

Change "Move Window (bars)" to `50`.

Expected: the "Idio Z" histogram becomes much smoother and slower-moving, but its
typical amplitude stays in the same ±2-ish range — that is the `math.sqrt(lenMove)`
scaling doing its job. If the amplitude grows roughly fourfold instead, the square
root scaling has been dropped. Reset to `12`.

- [ ] **Step 4: Commit**

```bash
git add xau-attribution.pine
git commit -m "feat: orthogonalise silver and isolate gold-idiosyncratic residual"
```

---

### Task 5: Attribute the move window and derive the verdict

**Files:**
- Modify: `xau-attribution.pine` — replace the TEMPORARY plots from Task 4

**Interfaces:**
- Consumes: `gRet`, `dRet` from Task 2; `bD`, `r2D`, `sdG` from Task 3;
  `bS`, `r2S`, `sResid`, `idioZ` from Task 4; `lenReg`, `lenMove`,
  `minR2`, `dollarThreshold`, `complexThreshold`, `idioThreshold`,
  `compactMode`, `EPS` from Task 1.
- Produces: `moveTotal`, `moveDollar`, `moveSilver` (series float),
  `dollarShare`, `silverShare` (series float, uncapped, may exceed 1),
  `noOpinion` (series bool), `verdict` (series string), `verdictColor`
  (series color).

- [ ] **Step 1: Replace the temporary plots with the attribution and verdict**

Find this block at the end of the file:

```pine
// TEMPORARY — removed in Task 5
plot(idioZ, title="Idio Z", style=plot.style_histogram, color=color.fuchsia, linewidth=3)
plot(r2S, title="R2 silver", color=color.blue)
```

Replace it with:

```pine
// MOVE ATTRIBUTION
// nz() is correct for these sliding sums even though it is forbidden for the
// variance inputs above: a stale bar carried no new information, so it
// contributed no movement, and 0 is the honest contribution.

dollarPart = na(bD) or na(dRet) ? na : bD * dRet
silverPart = na(bS) or na(sResid) ? na : bS * sResid

moveTotal  = math.sum(nz(gRet), lenMove)
moveDollar = math.sum(nz(dollarPart), lenMove)
moveSilver = math.sum(nz(silverPart), lenMove)

// A move is "too small to attribute" relative to typical volatility over the
// same window, not against a fixed price threshold.
absTotal  = math.abs(moveTotal)
moveFloor = na(sdG) ? na : sdG * math.sqrt(lenMove) * 0.1
hasMove   = not na(absTotal) and not na(moveFloor) and absTotal > math.max(moveFloor, EPS)

// Shares are uncapped and are NOT expected to sum to 1. A share above 1 means
// components offset each other — a dollar leg lifting gold while the
// idiosyncratic leg pushed it down harder. That is a real state worth seeing.
dollarShare = hasMove ? math.abs(moveDollar) / absTotal : na
silverShare = hasMove ? math.abs(moveSilver) / absTotal : na

// VERDICT

warm = bar_index >= lenReg + lenMove

noOpinion = not warm or na(r2D) or na(r2S) or na(idioZ) or not hasMove or (r2D < minR2 and r2S < minR2)

verdict = noOpinion ? "NO OPINION" :
     dollarShare > dollarThreshold ? "DOLLAR-DRIVEN" :
     silverShare > complexThreshold ? "COMPLEX-WIDE" :
     math.abs(idioZ) > idioThreshold ? "GOLD-IDIOSYNCRATIC" : "MIXED"

verdictColor = switch verdict
    "NO OPINION"         => color.new(color.gray, 55)
    "DOLLAR-DRIVEN"      => color.new(color.orange, 15)
    "COMPLEX-WIDE"       => color.new(color.blue, 15)
    "GOLD-IDIOSYNCRATIC" => color.new(color.fuchsia, 0)
    => color.new(color.gray, 25)

// PLOT

plot(compactMode ? na : idioZ, title="Idiosyncratic Z", style=plot.style_histogram, color=verdictColor, linewidth=3)

hline(0, "Zero", color=color.new(color.gray, 50))

// Threshold guides are plots, not hlines. `hline` requires an input-qualified
// value; `-idioThreshold` is arithmetic on an input, which demotes it to
// "simple" and fails to compile. plot() has no such restriction.
plot(idioThreshold,  title="Upper threshold", color=color.new(color.gray, 70), style=plot.style_linebr)
plot(-idioThreshold, title="Lower threshold", color=color.new(color.gray, 70), style=plot.style_linebr)
```

- [ ] **Step 2: Verify the histogram is coloured by verdict**

Follow the **Verification method** above on `OANDA:XAUUSD`, 15-minute.

Expected:

- The histogram now changes colour along its length rather than being uniformly
  fuchsia. Grey bars (NO OPINION) appear over the warmup region at the far left
  and in quiet stretches; orange, blue, fuchsia and mid-grey bars appear
  elsewhere.
- Two dashed horizontal lines sit at +1.5 and −1.5, and a solid line at 0.
- Fuchsia bars (GOLD-IDIOSYNCRATIC) occur only outside the dashed lines.

- [ ] **Step 3: Verify the R² gate suppresses the verdict**

Set "Minimum R² for an Opinion" to `0.95`.

Expected: almost the entire histogram turns grey, because the R² gate now
rejects nearly every bar. Set it to `0.0`.

Expected: no bars are grey for the R² reason — only the warmup region and any
too-small-to-attribute stretches remain grey. Reset to `0.15`.

- [ ] **Step 4: Verify the dollar threshold takes effect**

Set "Dollar-Driven Threshold" to `0.05`.

Expected: the histogram becomes overwhelmingly orange, since almost any dollar
contribution now clears the bar. Reset to `0.6`.

- [ ] **Step 5: Verify compact mode hides the histogram**

Tick "Compact Mode".

Expected: the histogram disappears entirely; the three horizontal reference lines
remain. Untick it.

- [ ] **Step 6: Commit**

```bash
git add xau-attribution.pine
git commit -m "feat: attribute the move window and derive the verdict"
```

---

### Task 6: Add the readout table

**Files:**
- Modify: `xau-attribution.pine` (append after the PLOT block)

**Interfaces:**
- Consumes: `verdict`, `verdictColor`, `dollarShare`, `silverShare` from Task 5;
  `idioZ`, `r2D` from Tasks 3–4; `tablePos`, `tableSize`, `compactMode` from Task 1.
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Append the formatting helpers and the table**

Append the following to the end of `xau-attribution.pine`:

```pine

// TABLE
// Built on the last bar only. A table is not a line/box/label, so it does not
// consume the per-candle drawing budget.

fmtPct(v) =>
    na(v) ? "—" : str.tostring(v * 100, "#.#") + "%"

fmtNum(v) =>
    na(v) ? "—" : str.tostring(v, "#.##")

var table infoTable = table.new(tablePos, 2, 5, border_width = 1)

if barstate.islast
    headerBg = color.new(color.black, 20)
    rowBg    = color.new(color.black, 40)
    labelCol = color.new(color.white, 30)

    table.cell(infoTable, 0, 0, "XAU Attribution", text_color = labelCol, bgcolor = headerBg, text_size = tableSize)
    table.cell(infoTable, 1, 0, verdict, text_color = verdictColor, bgcolor = headerBg, text_size = tableSize)

    if not compactMode
        table.cell(infoTable, 0, 1, "$ share",  text_color = labelCol, bgcolor = rowBg, text_size = tableSize)
        table.cell(infoTable, 1, 1, fmtPct(dollarShare), text_color = labelCol, bgcolor = rowBg, text_size = tableSize)

        table.cell(infoTable, 0, 2, "Ag share", text_color = labelCol, bgcolor = rowBg, text_size = tableSize)
        table.cell(infoTable, 1, 2, fmtPct(silverShare), text_color = labelCol, bgcolor = rowBg, text_size = tableSize)

        table.cell(infoTable, 0, 3, "Idio z",   text_color = labelCol, bgcolor = rowBg, text_size = tableSize)
        table.cell(infoTable, 1, 3, fmtNum(idioZ), text_color = labelCol, bgcolor = rowBg, text_size = tableSize)

        table.cell(infoTable, 0, 4, "R² $",     text_color = labelCol, bgcolor = rowBg, text_size = tableSize)
        table.cell(infoTable, 1, 4, fmtNum(r2D), text_color = labelCol, bgcolor = rowBg, text_size = tableSize)
```

- [ ] **Step 2: Verify the table renders with live values**

Follow the **Verification method** above on `OANDA:XAUUSD`, 15-minute.

Expected:

- A five-row table in the top right of the **indicator pane**, headed "XAU
  Attribution" with the verdict beside it in the same colour as the current
  histogram bar.
- Rows read "$ share", "Ag share", "Idio z", "R² $" with numeric values. Shares
  show as percentages, `Idio z` and `R² $` to two decimal places.
- `R² $` is between 0.00 and 1.00 and matches what the R² gate implies — if the
  verdict is NO OPINION on R² grounds, this value is below the threshold.

- [ ] **Step 3: Verify compact mode reduces the table to the verdict**

Tick "Compact Mode".

Expected: the histogram disappears and the table shrinks to a single row —
"XAU Attribution" and the verdict. No numeric rows remain. Untick it.

- [ ] **Step 4: Verify the table position and size inputs**

Set "Table Position" to `Bottom Left`.

Expected: the table moves to the bottom-left of the pane. Set "Table Text Size"
to `Large`.

Expected: the text grows. Reset both to `Top Right` and `Small`.

- [ ] **Step 5: Verify the em-dash fallback for undefined values**

Scroll the chart back so the leftmost visible bar is inside the warmup region,
then use the keyboard left arrow to move the cursor into it.

Expected: with no valid regression, the verdict reads "NO OPINION". (The table
itself renders on the last bar regardless of cursor position; this step is
checking that `fmtPct` / `fmtNum` never print `NaN` — if the table shows `NaN`
anywhere at any time, the helpers are wrong.)

- [ ] **Step 6: Commit**

```bash
git add xau-attribution.pine
git commit -m "feat: add XAU attribution readout table with compact mode"
```

---

### Task 7: Document the indicator in the README

**Files:**
- Modify: `README.md` — the `## Indicators` section

**Interfaces:**
- Consumes: the finished `xau-attribution.pine`.
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Add a section after "Simple Sessions"**

In `README.md`, find this block:

```markdown
### Simple Sessions
Highlights global trading sessions (US, EU, Asia/Tokyo, and a user-defined session) as background fills on the chart.

**Features:**
- Toggle each session on/off with customizable fill colors
- Accounts for daylight saving time across regions
- Shows session high/low lines
- Optional user-defined custom session with timezone support
```

Insert the following immediately after it (leaving a blank line between the two
sections):

```markdown
### XAU Attribution
Decomposes each XAUUSD move into a dollar leg, a precious-metals-complex leg, and a gold-idiosyncratic residual, so a sweep can be checked for an external driver before it is faded.

**Features:**
- Rolling regression against a dollar index and silver, using sequential orthogonalisation so the two factors do not double-count
- Plain-English verdict — Dollar-driven, Complex-wide, Gold-idiosyncratic, Mixed, or No opinion
- Reports R² and declines to give a verdict when the correlations are not live
- Idiosyncratic extension shown as a z-score histogram, coloured by verdict
- Configurable dollar and silver symbols, regression lookback, and move window
- Compact mode reduces the display to a single verdict line for an execution chart

Note: this is a filter, not a signal. It is intended to rule out fades that are really dollar moves, not to generate entries on its own.
```

- [ ] **Step 2: Verify the README renders correctly**

Run:

```bash
sed -n '/### XAU Attribution/,/^## /p' README.md
```

Expected: the new section prints in full, with its heading, description,
"**Features:**" line, six bullets, and the closing note.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: document XAU Attribution indicator in README"
```

---

## Notes for the implementer

- **Why beta is computed from correlation and standard deviations.** The OLS
  slope of y on x equals `corr(x,y) * sd(y) / sd(x)`. Pine has `ta.correlation`
  and `ta.stdev` as built-ins, so the whole regression is three function calls
  with no loop. This matters beyond elegance: a manual `for` loop over the window
  would need guarding, because `for i = 0 to n - 1` counts *downward* when `n` is
  0 rather than skipping.

- **Why the silver step regresses residual on residual.** Silver and the dollar
  are correlated with each other. Regressing gold on each independently would let
  both claim the shared component, and the parts would sum to more than the
  whole. Removing the dollar from silver first makes the second factor orthogonal
  to the first, so the three components are additive. This is the Frisch–Waugh–Lovell
  theorem, and it is what makes the "$ share" and "Ag share" numbers comparable.

- **Why `nz()` appears in the sums but never in the variances.** Replacing a
  missing return with 0 in a variance calculation understates the true dispersion
  and inflates R², which would make the indicator confidently wrong exactly when
  data is thin. In a sliding *sum* over the move window, 0 is the honest
  contribution of a bar that carried no new information. The asymmetry is
  deliberate; do not "tidy" it into consistency.

- **Why `request.security` needs the bar time.** It forward-fills. On a bar where
  the reference symbol did not print, it returns the previous close, and the
  computed return is exactly 0 — indistinguishable from a genuine flat bar but
  statistically corrupting. Comparing the returned bar `time` against its previous
  value is what separates the two cases.

- **Shares can exceed 100%, and that is not a bug.** When components offset each
  other the absolute contribution of one can be larger than the absolute net move.
  Clamping the display would hide precisely the situations most worth seeing.

- **The thresholds are guesses.** `minR2 = 0.15`, `dollarThreshold = 0.6`,
  `complexThreshold = 0.3` and `idioThreshold = 1.5` were chosen to be
  approximately right, not derived from data. They are exposed as inputs
  specifically so they can be retuned against live charts, and it would be
  reasonable for all four to change after the first week of use.

- **What this indicator does not do.** It has no useful read during a high-impact
  release, when both legs move on the same news — the decomposition stays
  arithmetically valid but the move is repricing, neither a grab nor a fade.
  In low volatility both reference legs are noise. It is informative around
  impulsive moves and level interactions, not continuously.
