# Previous Day Midpoint — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create `prior-day-midpoint.pine`, an indicator that draws the prior day's midpoint as a light grey, labelled horizontal segment across each day, projected forward to the current day's session close.

**Architecture:** A single new `.pine` file. One `request.security` call on the daily timeframe supplies the previous day's high/low plus the current day's open and close timestamps. On each new day a time-anchored `line` (and optional `label`) is created spanning that day's open to close at the previous day's midpoint. Handles are held in two parallel arrays, pruned oldest-first to the configured display count, and every label's text is rewritten at each day boundary so the `D-n M` naming stays relative to today.

**Tech Stack:** Pine Script v6, TradingView indicator. No test framework — verification is done by loading the script in the TradingView Pine Editor and inspecting behaviour on an intraday chart.

## Global Constraints

- Pine Script v6 (`//@version=6`).
- File begins with the MPL 2.0 comment line used by the other scripts in this repo.
- Indicator title `"Previous Day Midpoint"`, shorttitle `"D-1 M"`, `overlay=true`.
- Default colour for both line and label text is `#B0B0B0`.
- Drawings render only when `timeframe.in_seconds() <= 86400` (daily and below).
- Never reference future price data: price comes from `high[1]` / `low[1]` on the
  daily series under `lookahead=barmerge.lookahead_on`, the same non-repainting
  idiom already used in `prior-day-ohlcm.pine`.
- All drawings are time-anchored (`xloc.bar_time`) so they can extend past the
  last bar into future space.

## Verification method

There is no automated test runner for Pine Script. Every task that changes
`prior-day-midpoint.pine` is verified the same way:

1. Open <https://www.tradingview.com/> and open the Pine Editor.
2. Paste the full contents of `prior-day-midpoint.pine` into the editor.
3. Click **Add to chart**.
4. Confirm there is no red compilation error in the editor's console.
5. Inspect the chart against the task's stated expectation.

Use a liquid 24-hour symbol on a **5-minute** chart for inspection (for example
`BINANCE:BTCUSDT` or `OANDA:EURUSD`), because a 24h symbol makes the day
boundaries and the forward projection easy to see.

---

### Task 1: Create the script skeleton and all inputs

**Files:**
- Create: `prior-day-midpoint.pine`

**Interfaces:**
- Consumes: nothing.
- Produces: input variables `daysToShow` (int), `lineColorInput` (color),
  `lineWidthInput` (int), `lineStyleInput` (string), `showLabels` (bool),
  `labelPosInput` (string), `labelColorInput` (color), `labelSizeInput`
  (string); and the derived constants `lineStyle` and `labelSize`.

- [ ] **Step 1: Create the file with the header, declaration, inputs and style mapping**

Create `prior-day-midpoint.pine` with exactly this content:

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/

//@version=6
indicator("Previous Day Midpoint", shorttitle="D-1 M", overlay=true, max_lines_count=500, max_labels_count=500)

// USER SETTINGS

daysToShow = input.int(2, title="Number of Days to Display", minval=1, maxval=100,
     tooltip="Number of midpoint segments to draw, including the segment covering today.")

lineColorInput = input.color(#B0B0B0, title="Line Colour", group="Line")
lineWidthInput = input.int(1, title="Line Width", minval=1, maxval=10, group="Line")
lineStyleInput = input.string("Solid", title="Line Style", options=["Solid", "Dotted", "Dashed"], group="Line")

showLabels      = input.bool(true, title="Show Labels", group="Label")
labelPosInput   = input.string("Middle", title="Label Position", options=["Left", "Middle", "Right"], group="Label")
labelColorInput = input.color(#B0B0B0, title="Label Colour", group="Label")
labelSizeInput  = input.string("Small", title="Label Size", options=["Tiny", "Small", "Normal", "Large", "Huge"], group="Label")

// STYLE MAPPING

lineStyle = switch lineStyleInput
    "Solid"  => line.style_solid
    "Dotted" => line.style_dotted
    "Dashed" => line.style_dashed
    => line.style_solid

labelSize = switch labelSizeInput
    "Tiny"   => size.tiny
    "Small"  => size.small
    "Normal" => size.normal
    "Large"  => size.large
    "Huge"   => size.huge
    => size.small
```

- [ ] **Step 2: Verify it compiles and the settings dialog is correct**

Follow the **Verification method** above.

Expected: no compilation error. The indicator adds to the chart and draws
nothing. Opening its settings shows, in order: "Number of Days to Display"
(2), a "Line" group with Colour / Width / Style, and a "Label" group with Show
Labels / Label Position / Label Colour / Label Size.

- [ ] **Step 3: Commit**

```bash
git add prior-day-midpoint.pine
git commit -m "feat: add prior day midpoint indicator skeleton and inputs"
```

---

### Task 2: Draw one midpoint line per day

**Files:**
- Modify: `prior-day-midpoint.pine` (append after the STYLE MAPPING block)

**Interfaces:**
- Consumes: `daysToShow`, `lineColorInput`, `lineWidthInput`, `lineStyle` from Task 1.
- Produces: `tfAllowed` (bool), `prevMid` (series float), `dayStart` /
  `dayEnd` (series int, ms timestamps), `newDay` (series bool), and
  `midLines` (a `var line[]` holding the drawn segments, oldest at index 0,
  newest at the end).

- [ ] **Step 1: Append the guard, data, storage and drawing blocks**

Append the following to the end of `prior-day-midpoint.pine`:

```pine

// TIMEFRAME GUARD — daily and below only

tfAllowed = timeframe.in_seconds() <= 86400

// DATA
// high[1]/low[1] are the previous *completed* daily bar; time/time_close are the
// current daily bar's open and close timestamps.

[prevHigh, prevLow, dayStart, dayEnd] = request.security(syminfo.tickerid, "D",
     [high[1], low[1], time, time_close], lookahead=barmerge.lookahead_on)

prevMid = (prevHigh + prevLow) / 2

newDay = na(dayStart[1]) or dayStart != dayStart[1]

// STORAGE — oldest at index 0, newest at the end

var line[] midLines = array.new<line>()

// DRAWING

if tfAllowed and newDay and not na(prevMid) and not na(dayStart) and not na(dayEnd)
    if array.size(midLines) >= daysToShow
        oldLine = array.shift(midLines)
        if not na(oldLine)
            line.delete(oldLine)

    array.push(midLines, line.new(x1=dayStart, y1=prevMid, x2=dayEnd, y2=prevMid,
         xloc=xloc.bar_time, color=lineColorInput, width=lineWidthInput, style=lineStyle))
```

- [ ] **Step 2: Verify the segments draw and project correctly**

Follow the **Verification method** above on a 5-minute chart.

Expected, with the default "Number of Days to Display" = 2:

- Exactly two light grey horizontal segments are visible.
- The right-hand segment spans today's open to today's close. Because today is
  incomplete, it extends past the last bar into empty space on the right.
- The left-hand segment spans yesterday's open to yesterday's close only. The
  two segments do not overlap horizontally.
- The right-hand segment's price equals yesterday's `(high + low) / 2`. Check
  this by switching to a Daily chart, reading yesterday's high and low from the
  OHLC legend, averaging them, and comparing.

- [ ] **Step 3: Verify the display count input**

Change "Number of Days to Display" to `5`.

Expected: five non-overlapping segments, one per day, covering the last five
days including today.

- [ ] **Step 4: Verify the timeframe guard**

Switch the chart to the Weekly timeframe.

Expected: no segments are drawn at all. Switching back to 5-minute restores
them.

- [ ] **Step 5: Verify the line style inputs**

In settings, set Line Style to `Dashed` and Line Width to `3`.

Expected: the segments redraw as thick dashed lines. Reset to `Solid` / `1`.

- [ ] **Step 6: Commit**

```bash
git add prior-day-midpoint.pine
git commit -m "feat: draw prior day midpoint segments projected to session close"
```

---

### Task 3: Add relative `D-n M` labels

**Files:**
- Modify: `prior-day-midpoint.pine` — the STORAGE and DRAWING blocks added in Task 2

**Interfaces:**
- Consumes: `showLabels`, `labelPosInput`, `labelColorInput`, `labelSize` from
  Task 1; `tfAllowed`, `prevMid`, `dayStart`, `dayEnd`, `newDay`, `midLines`
  from Task 2.
- Produces: `labelX` (series int) and `midLabels` (a `var label[]` kept parallel
  to `midLines` whenever `showLabels` is true).

- [ ] **Step 1: Add the label array to the STORAGE block**

Find this block:

```pine
// STORAGE — oldest at index 0, newest at the end

var line[] midLines = array.new<line>()
```

Replace with:

```pine
// STORAGE — oldest at index 0, newest at the end

var line[]  midLines  = array.new<line>()
var label[] midLabels = array.new<label>()
```

- [ ] **Step 2: Add the label x-position mapping**

Insert this immediately after the STORAGE block and before the `// DRAWING`
comment:

```pine

// LABEL POSITION — a timestamp within the segment's day

labelX = switch labelPosInput
    "Left"   => dayStart
    "Middle" => dayStart + int((dayEnd - dayStart) / 2)
    "Right"  => dayEnd
    => dayStart + int((dayEnd - dayStart) / 2)
```

- [ ] **Step 3: Extend the drawing block with label creation, pruning and relabelling**

Find the whole DRAWING block:

```pine
if tfAllowed and newDay and not na(prevMid) and not na(dayStart) and not na(dayEnd)
    if array.size(midLines) >= daysToShow
        oldLine = array.shift(midLines)
        if not na(oldLine)
            line.delete(oldLine)

    array.push(midLines, line.new(x1=dayStart, y1=prevMid, x2=dayEnd, y2=prevMid,
         xloc=xloc.bar_time, color=lineColorInput, width=lineWidthInput, style=lineStyle))
```

Replace with:

```pine
if tfAllowed and newDay and not na(prevMid) and not na(dayStart) and not na(dayEnd)
    if array.size(midLines) >= daysToShow
        oldLine = array.shift(midLines)
        if not na(oldLine)
            line.delete(oldLine)
        if array.size(midLabels) > 0
            oldLabel = array.shift(midLabels)
            if not na(oldLabel)
                label.delete(oldLabel)

    array.push(midLines, line.new(x1=dayStart, y1=prevMid, x2=dayEnd, y2=prevMid,
         xloc=xloc.bar_time, color=lineColorInput, width=lineWidthInput, style=lineStyle))

    if showLabels
        array.push(midLabels, label.new(x=labelX, y=prevMid, xloc=xloc.bar_time, yloc=yloc.price,
             text="D-1 M", style=label.style_none, textcolor=labelColorInput, size=labelSize))

    // Labels are relative to today, so every stored label is renamed as it ages.
    // Newest is at the end of the array, so index i has age (labelCount - i).
    labelCount = array.size(midLabels)
    if labelCount > 0
        for i = 0 to labelCount - 1
            lb = array.get(midLabels, i)
            if not na(lb)
                label.set_text(lb, "D-" + str.tostring(labelCount - i) + " M")
```

- [ ] **Step 4: Verify labels render and are numbered relative to today**

Follow the **Verification method** above on a 5-minute chart, with "Number of
Days to Display" set to `4`.

Expected:

- Four segments, each with plain grey text sitting on the line — no label
  bubble, no arrow.
- The segment covering today reads `D-1 M`. Moving left, the next reads
  `D-2 M`, then `D-3 M`, then `D-4 M`.
- Each label sits at the horizontal centre of its segment.

- [ ] **Step 5: Verify the label inputs**

- Set Label Position to `Left`. Expected: each label moves to the left end of
  its segment. Set it to `Right`. Expected: each moves to the right end — the
  `D-1 M` label lands in the empty future space at today's close. Reset to
  `Middle`.
- Set Label Size to `Huge`. Expected: text grows. Reset to `Small`.
- Set Label Colour to red. Expected: text turns red, lines stay grey. Reset to
  `#B0B0B0`.
- Untick "Show Labels". Expected: all text disappears, all segments remain.
  Re-tick it.

- [ ] **Step 6: Commit**

```bash
git add prior-day-midpoint.pine
git commit -m "feat: add relative D-n M labels to prior day midpoint segments"
```

---

### Task 4: Document the indicator in the README

**Files:**
- Modify: `README.md` — the `## Indicators` section

**Interfaces:**
- Consumes: the finished `prior-day-midpoint.pine`.
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Add a section after "Previous Day OHLCM"**

In `README.md`, find this block:

```markdown
### Previous Day OHLCM
Displays the prior day's Open, High, Low, Close, and Midpoint as step lines on your chart.

**Features:**
- Shows all five prior-day levels (O, H, L, C, Mid) as persistent reference lines
- Step line style for clean horizontal level display
- Toggle visibility on/off
```

Insert the following immediately after it (leaving a blank line between the two
sections):

```markdown
### Previous Day Midpoint
Draws the prior day's midpoint — `(high + low) / 2` — as a labelled horizontal segment across each day, on any daily or intraday chart.

**Features:**
- One non-overlapping segment per day, each showing the previous day's midpoint
- Today's segment is projected forward to the session close, ahead of price
- Labels are relative to today (`D-1 M`, `D-2 M`, …) and renumber as days pass
- Configurable number of days to display
- Configurable line colour, width, and style (solid/dotted/dashed)
- Configurable label visibility, position, colour, and size
```

- [ ] **Step 2: Verify the README renders correctly**

Run:

```bash
sed -n '/### Previous Day Midpoint/,/^### /p' README.md
```

Expected: the new section prints in full, with its heading, description,
"**Features:**" line, and six bullets.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: document Previous Day Midpoint indicator in README"
```

---

## Notes for the implementer

- **Why `xloc.bar_time` and not bar indices.** A bar-index-anchored drawing
  cannot be placed reliably in future space without knowing how many bars fit in
  the remainder of the day. Timestamps are exact and session-aware.
- **Why the segment is drawn once, complete.** Both endpoints are known the
  moment the day opens (`dayStart` and `dayEnd` both come from the daily
  series), so there is no need to extend `x2` on every bar the way
  `prev-htf-high-low.pine` does.
- **Why `for` loops are guarded with `if labelCount > 0`.** In Pine,
  `for i = 0 to -1` iterates downward rather than skipping, so an unguarded loop
  over an empty array would run with a negative index and raise a runtime error.
- **First day of history.** `high[1]` on the daily series is `na` there, so
  `prevMid` is `na` and the `not na(prevMid)` guard suppresses that segment.
  This is expected, not a bug.
