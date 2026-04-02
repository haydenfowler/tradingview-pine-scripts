# Previous HTF High/Low — Design Spec

## Overview

A Pine Script v6 overlay indicator that draws horizontal lines at the previous higher-timeframe period's high and low prices. Lines span from the period they represent through the next period. The number of visible level pairs is determined dynamically by how many selected-timeframe periods fit into the next-higher timeframe.

## Indicator Setup

- Pine Script v6, `overlay=true`
- Name: `"Previous HTF High/Low"`
- `max_lines_count=500`

## User Settings

| Setting | Type | Default | Options / Range |
|---------|------|---------|-----------------|
| Timeframe | `input.string` | `"D"` | `["Auto", "1m", "5m", "15m", "30m", "1h", "4h", "D", "W", "M"]` |
| Line Color | `input.color` | `#FFA500` (orange) | Any color |
| Line Thickness | `input.int` | `1` | 1–10 |
| Line Style | `input.string` | `"Solid"` | `["Solid", "Dotted", "Dashed"]` |

### Timeframe helpers

Reuse the same pattern from `hft-candles.pine`:

- `get_timeframe_string(tf_input)` — converts user-friendly labels (`"1h"` → `"60"`, etc.)
- `get_next_higher_timeframe()` — returns the next timeframe up from the chart timeframe (used when `"Auto"` is selected)

Default is `"D"` (not `"Auto"` like hft-candles).

## Core Logic

### Data sourcing

Use `request.security()` to fetch `high` and `low` from the selected timeframe. This gives us the aggregated high/low for each period on that timeframe.

### Period change detection

Use `timeframe.change(selected_timeframe)` to detect when a new period begins. On each new period, the just-completed period's high and low are stored.

### Display count

The number of high/low pairs to display is computed dynamically:

```
display_count = timeframe.in_seconds(next_higher_tf) / timeframe.in_seconds(selected_tf)
```

Where `next_higher_tf` is determined by the same step-up logic used in `get_next_higher_timeframe()`, but applied to the *selected* timeframe rather than the chart timeframe.

Examples:
- Selected `"D"`, next higher `"W"` → `604800 / 86400 = 7` (but typically 5 trading days visible)
- Selected `"1h"` (`"60"`), next higher `"4h"` (`"240"`) → `14400 / 3600 = 4`
- Selected `"15m"`, next higher `"1h"` → `3600 / 900 = 4`

### Storage

Arrays store the historical data for rendering:
- `float[] prev_highs` — high prices of completed periods
- `float[] prev_lows` — low prices of completed periods
- `int[] period_start_bars` — bar_index at the start of each period
- `int[] period_end_bars` — bar_index at the end of each period (updated as bars arrive)
- `line[] high_lines` — line objects for highs
- `line[] low_lines` — line objects for lows

Arrays are capped at `display_count` entries; oldest entries are shifted out and their line objects deleted.

### Line rendering

Each high/low pair produces two horizontal lines via `line.new()`:
- **x1**: First bar of the period the level belongs to
- **x2**: Last bar of the *next* period (extends one full period forward)
- **y1 = y2**: The high or low price (horizontal)
- Color, thickness, and style from user settings
- Both high and low lines use identical styling

Lines for the current (in-progress) period are updated on each bar to extend `x2` to the current `bar_index`.

### Guard

Only render when the chart timeframe is lower than the selected timeframe (`timeframe.in_seconds() < timeframe.in_seconds(selected_tf)`).

## What This Does NOT Include

- Labels or text on lines
- Opacity fading for older levels
- Different styles for high vs low lines
- Fill/shading between high and low
