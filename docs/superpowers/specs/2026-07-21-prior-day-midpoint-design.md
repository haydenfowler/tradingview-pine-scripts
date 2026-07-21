# Previous Day Midpoint — Design

**Date:** 2026-07-21
**File:** `prior-day-midpoint.pine`
**Status:** Approved

## Purpose

Plot the prior day's midpoint — `(high + low) / 2` — as a light grey horizontal
level, so the level is visible on any intraday chart without switching
timeframes. Unlike `prior-day-ohlcm.pine`, which draws a continuous step line for
five levels, this indicator draws one discrete, labelled segment per day and
projects the current day's segment forward to the session close.

## Behaviour

### Segment geometry

One segment per day. Each segment sits at the **previous** day's midpoint and
spans the **following** day's open to that day's close. Segments do not overlap.

```
D-3      D-2      D-1      D0 (today)
         ────────                        <- "D-3 M"
                  ────────               <- "D-2 M"
                           ────────      <- "D-1 M"
```

The segment covering today is drawn across the whole day at the moment the day
opens, projecting into empty future space to the right of the last bar. It is
static once created; it does not grow bar by bar.

### Labels

Labels are relative to today, not to the segment's own day:

- The segment covering today reads `D-1 M` (it shows the mid of one day ago).
- The segment covering yesterday reads `D-2 M`.
- …and so on.

Because the labels are relative, every stored label's text is rewritten at each
day boundary as the segments age.

Default label position is the horizontal centre of the segment, configurable to
the left or right end. Labels render with `label.style_none` — text only, no
bubble or arrow — so they sit cleanly on the line.

### Display count

The `Number of days to display` input is the number of segments on the chart,
including today's. `2` means today's segment (`D-1 M`) plus yesterday's
(`D-2 M`). Older segments are deleted as new days open.

### Timeframe guard

The indicator draws only when `timeframe.in_seconds() <= 86400` — daily and
below. On weekly and monthly charts nothing is drawn. On a daily chart each
segment is naturally one bar wide.

## Implementation

### Data

A single security call supplies everything needed:

```pine
[prevHigh, prevLow, dayStart, dayEnd] = request.security(
     syminfo.tickerid, "D", [high[1], low[1], time, time_close],
     lookahead = barmerge.lookahead_on)
```

- `high[1]` / `low[1]` are the previous *completed* daily bar's extremes.
  Referencing `[1]` under `lookahead_on` is the standard non-repainting idiom,
  already used by `prior-day-ohlcm.pine`.
- `time` is the current daily bar's opening timestamp.
- `time_close` is the current daily bar's closing timestamp — the session's
  known end, so no assumption about midnight versus an RTH close is needed.

The midpoint is `(prevHigh + prevLow) / 2`.

### Day detection

A new day is detected by `dayStart != dayStart[1]`. This reuses a value the
script already needs, avoiding a second security call.

### Drawing

Lines and labels are anchored with `xloc.bar_time`. Time anchoring is what
allows a drawing to extend past the last bar into future space; bar-index
anchoring would be limited and would need the bar spacing inferred.

Each segment is created once, on the first chart bar of its day, with both
endpoints already known (`dayStart` → `dayEnd`). No per-bar endpoint updates.

### Storage and pruning

Two parallel arrays hold the drawing handles:

```pine
var line[]  midLines  = array.new<line>()
var label[] midLabels = array.new<label>()
```

On each new day, before pushing the new segment: if the arrays are at capacity,
`array.shift` the oldest handles off and `line.delete` / `label.delete` them.
The indicator declares `max_lines_count = 500` and `max_labels_count = 500`.

Newest segment is at the end of the array. Relabelling walks the array from the
end backwards, assigning `D-1 M`, `D-2 M`, … in order.

### Reacting to input changes

Pine re-runs the whole script when an input changes, so no special handling is
needed for a change to the display count, colour, or label settings.

## Inputs

| Input | Type | Default |
|---|---|---|
| Number of days to display | int, 1–100 | 2 |
| Line colour | color | `#B0B0B0` |
| Line width | int, 1–10 | 1 |
| Line style | Solid / Dotted / Dashed | Solid |
| Show labels | bool | true |
| Label position | Left / Middle / Right | Middle |
| Label colour | color | `#B0B0B0` |
| Label size | Tiny / Small / Normal / Large / Huge | Small |

The Solid/Dotted/Dashed string-to-constant mapping follows the `switch` pattern
already established in `prev-htf-high-low.pine`.

## Edge cases

- **First day of loaded history** — `prevHigh` / `prevLow` are `na`, so the
  midpoint is `na` and no segment is created for that day.
- **Timeframe above daily** — the guard suppresses all drawing.
- **Daily chart** — segments are one bar wide; labels still render.
- **Symbols with a non-24h session** — `time_close` from the daily series gives
  the true session close, so segments terminate at the close rather than
  overrunning to midnight.
- **Display count of 1** — only today's `D-1 M` segment is drawn.

## Out of scope

- Any level other than the midpoint (high, low, open, close) — those are
  covered by `prior-day-ohlcm.pine`.
- Higher-timeframe midpoints (weekly, monthly).
- Alerts on midpoint touches.

## Documentation

Add a "Previous Day Midpoint" section to `README.md`, following the format of
the existing indicator entries.
