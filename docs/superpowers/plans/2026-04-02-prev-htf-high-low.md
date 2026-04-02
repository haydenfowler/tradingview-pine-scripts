# Previous HTF High/Low Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a Pine Script v6 indicator that draws horizontal lines at the previous higher-timeframe period's high and low prices.

**Architecture:** Single-file Pine Script indicator using `request.security()` for HTF data, `timeframe.change()` for period detection, and arrays to manage a rolling window of line objects. Display count is dynamically computed from the ratio of the next-higher timeframe to the selected timeframe.

**Tech Stack:** Pine Script v6, TradingView

**Spec:** `docs/superpowers/specs/2026-04-02-prev-htf-high-low-design.md`

---

### Task 1: Indicator shell with user settings

**Files:**
- Create: `prev-htf-high-low.pine`

- [ ] **Step 1: Create the indicator file with header, settings, and timeframe helpers**

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/

//@version=6
indicator("Previous HTF High/Low", overlay=true, max_lines_count=500)

// USER SETTINGS

timeframe_input = input.string("D", title="Timeframe",
                               options=["Auto", "1m", "5m", "15m", "30m", "1h", "4h", "D", "W", "M"],
                               tooltip="Select 'Auto' to automatically use one timeframe higher than the chart, or choose a specific timeframe")

line_color_input = input.color(#FFA500, title="Line Color")
line_thickness_input = input.int(1, title="Line Thickness", minval=1, maxval=10)
line_style_input = input.string("Solid", title="Line Style", options=["Solid", "Dotted", "Dashed"])

// TIMEFRAME HELPERS

get_timeframe_string(tf_input) =>
    switch tf_input
        "1m" => "1"
        "5m" => "5"
        "15m" => "15"
        "30m" => "30"
        "1h" => "60"
        "4h" => "240"
        => tf_input  // "Auto", "D", "W", "M" pass through unchanged

get_next_higher_timeframe() =>
    chart_tf = timeframe.in_seconds() / 60
    result = switch
        chart_tf < 1 => "1"
        chart_tf < 5 => "5"
        chart_tf < 15 => "15"
        chart_tf < 30 => "30"
        chart_tf < 60 => "60"
        chart_tf < 240 => "240"
        chart_tf < 1440 => "D"
        chart_tf < 10080 => "W"
        => "M"
    result

selected_tf = timeframe_input == "Auto" ? get_next_higher_timeframe() : get_timeframe_string(timeframe_input)

line_style = switch line_style_input
    "Solid" => line.style_solid
    "Dotted" => line.style_dotted
    "Dashed" => line.style_dashed
    => line.style_solid
```

- [ ] **Step 2: Verify it compiles**

Paste into TradingView Pine Editor and click "Add to chart." The indicator should appear in the indicator list with no visual output yet. Settings panel should show all four inputs with correct defaults.

- [ ] **Step 3: Commit**

```bash
git add prev-htf-high-low.pine
git commit -m "feat: add indicator shell with user settings and timeframe helpers"
```

---

### Task 2: Display count calculation and guard

**Files:**
- Modify: `prev-htf-high-low.pine`

- [ ] **Step 1: Add the next-higher-timeframe-for-selected function and display count logic**

Append after the `line_style` variable:

```pine
// DISPLAY COUNT
// How many periods of the selected TF fit into the next-higher TF

get_next_higher_for_tf(tf_seconds) =>
    tf_minutes = tf_seconds / 60
    result = switch
        tf_minutes < 1 => "1"
        tf_minutes < 5 => "5"
        tf_minutes < 15 => "15"
        tf_minutes < 30 => "30"
        tf_minutes < 60 => "60"
        tf_minutes < 240 => "240"
        tf_minutes < 1440 => "D"
        tf_minutes < 10080 => "W"
        => "M"
    result

selected_tf_seconds = timeframe.in_seconds(selected_tf)
next_higher_tf = get_next_higher_for_tf(selected_tf_seconds)
next_higher_tf_seconds = timeframe.in_seconds(next_higher_tf)
display_count = int(next_higher_tf_seconds / selected_tf_seconds)

// GUARD — only render when chart TF is lower than selected TF
chart_is_lower = timeframe.in_seconds() < selected_tf_seconds
```

- [ ] **Step 2: Add a debug label to verify display count**

Append temporarily:

```pine
if barstate.islast and chart_is_lower
    label.new(bar_index, high, str.tostring(display_count), color=color.new(color.black, 100), textcolor=line_color_input)
```

Add to chart. On a 1h chart with Daily selected, label should show `4` (4h is next above 1h... wait — the next higher above Daily is Weekly, so `7`). On a 15m chart with 1h selected, should show `4`. Verify a couple of combos.

- [ ] **Step 3: Remove the debug label and commit**

Delete the debug label code, then:

```bash
git add prev-htf-high-low.pine
git commit -m "feat: add display count calculation and chart guard"
```

---

### Task 3: Data sourcing and period tracking

**Files:**
- Modify: `prev-htf-high-low.pine`

- [ ] **Step 1: Add request.security calls and period change detection with array storage**

Append after the guard variables:

```pine
// DATA

[htf_high, htf_low] = request.security(syminfo.tickerid, selected_tf, [high, low])

new_period = timeframe.change(selected_tf)

// STORAGE

var float[] prev_highs = array.new<float>()
var float[] prev_lows = array.new<float>()
var int[] period_start_bars = array.new<int>()
var int[] period_end_bars = array.new<int>()
var line[] high_lines = array.new<line>()
var line[] low_lines = array.new<line>()

var float completed_high = na
var float completed_low = na
var int current_period_start = na

// PERIOD TRACKING

if chart_is_lower
    if new_period
        // Store the just-completed period (if we have data)
        if not na(completed_high) and not na(current_period_start)
            // Cap array size — delete oldest if at limit
            if array.size(prev_highs) >= display_count
                array.shift(prev_highs)
                array.shift(prev_lows)
                array.shift(period_start_bars)
                array.shift(period_end_bars)
                old_h_line = array.shift(high_lines)
                old_l_line = array.shift(low_lines)
                line.delete(old_h_line)
                line.delete(old_l_line)

            array.push(prev_highs, completed_high)
            array.push(prev_lows, completed_low)
            array.push(period_start_bars, current_period_start)
            array.push(period_end_bars, bar_index - 1)

            // Create placeholder lines (will be positioned in rendering step)
            array.push(high_lines, line.new(na, na, na, na))
            array.push(low_lines, line.new(na, na, na, na))

        // Start tracking the new period
        completed_high := htf_high
        completed_low := htf_low
        current_period_start := bar_index
    else
        // Update running high/low for current period
        if not na(htf_high)
            completed_high := htf_high
        if not na(htf_low)
            completed_low := htf_low
```

- [ ] **Step 2: Commit**

```bash
git add prev-htf-high-low.pine
git commit -m "feat: add HTF data sourcing and period tracking arrays"
```

---

### Task 4: Line rendering

**Files:**
- Modify: `prev-htf-high-low.pine`

- [ ] **Step 1: Add line rendering logic**

Replace the placeholder line creation inside the `if not na(completed_high)` block — change the two `array.push` calls for `high_lines` and `low_lines` from placeholder lines to real lines. Replace:

```pine
            // Create placeholder lines (will be positioned in rendering step)
            array.push(high_lines, line.new(na, na, na, na))
            array.push(low_lines, line.new(na, na, na, na))
```

With:

```pine
            array.push(high_lines, line.new(x1=current_period_start, y1=completed_high,
                                            x2=bar_index, y2=completed_high,
                                            color=line_color_input, width=line_thickness_input, style=line_style))
            array.push(low_lines, line.new(x1=current_period_start, y1=completed_low,
                                           x2=bar_index, y2=completed_low,
                                           color=line_color_input, width=line_thickness_input, style=line_style))
```

- [ ] **Step 2: Add line extension logic for the current period**

Append at the end of the `if chart_is_lower` block (after the period tracking code):

```pine
    // Extend the most recent completed period's lines to current bar
    n = array.size(high_lines)
    if n > 0
        last_h = array.get(high_lines, n - 1)
        last_l = array.get(low_lines, n - 1)
        if not na(last_h)
            line.set_x2(last_h, bar_index)
        if not na(last_l)
            line.set_x2(last_l, bar_index)
```

- [ ] **Step 3: Verify on TradingView**

Add to a 1h chart with Daily timeframe selected. You should see horizontal orange lines at the previous day's high and low, extending from the start of that day through the current bar. As you scroll back, older days should also have lines (up to `display_count` pairs).

- [ ] **Step 4: Commit**

```bash
git add prev-htf-high-low.pine
git commit -m "feat: add horizontal line rendering for previous HTF highs and lows"
```

---

### Task 5: Line extension through next period

**Files:**
- Modify: `prev-htf-high-low.pine`

The lines currently extend from their own period start to the current bar (for the most recent) or to the end of their own period (for older ones). Per spec, each line should span from its period's start through the *end of the next period*.

- [ ] **Step 1: Update period_end_bars for older entries when a new period starts**

Inside the `if new_period` block, after pushing the new entry to the arrays but before the "Start tracking the new period" comment, add logic to update the previous entry's end bar. Locate this section:

```pine
            array.push(high_lines, line.new(x1=current_period_start, y1=completed_high,
                                            x2=bar_index, y2=completed_high,
                                            color=line_color_input, width=line_thickness_input, style=line_style))
            array.push(low_lines, line.new(x1=current_period_start, y1=completed_low,
                                           x2=bar_index, y2=completed_low,
                                           color=line_color_input, width=line_thickness_input, style=line_style))

        // Start tracking the new period
```

And add between the push calls and the comment:

```pine
            // Update the second-to-last entry's lines to extend into this period
            entry_count = array.size(high_lines)
            if entry_count >= 2
                prev_h = array.get(high_lines, entry_count - 2)
                prev_l = array.get(low_lines, entry_count - 2)
                prev_end = array.get(period_end_bars, array.size(period_end_bars) - 2)
                // The previous entry should extend to the end of *this* entry's period
                // For now set to bar_index; it will be finalized when the next period starts
                if not na(prev_h)
                    line.set_x2(prev_h, bar_index - 1)
                if not na(prev_l)
                    line.set_x2(prev_l, bar_index - 1)
```

- [ ] **Step 2: Verify on TradingView**

On a 1h chart with Daily selected: each day's high/low lines should extend from that day's first bar through the end of the following day. The most recent completed period's lines extend to the current bar.

- [ ] **Step 3: Commit**

```bash
git add prev-htf-high-low.pine
git commit -m "feat: extend lines through the next period per spec"
```

---

### Task 6: Final cleanup and verification

**Files:**
- Modify: `prev-htf-high-low.pine`

- [ ] **Step 1: Add license header**

Ensure the file starts with:

```pine
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
```

- [ ] **Step 2: Full verification on TradingView**

Test these scenarios:
1. **1h chart, Daily timeframe** — should show ~7 days of high/low lines (daily → weekly ratio)
2. **15m chart, 1h timeframe** — should show 4 pairs (1h → 4h ratio)
3. **5m chart, Auto** — should auto-select 15m and show 2 pairs (15m → 30m ratio)
4. **Change color** to blue in settings — all lines update
5. **Change style** to Dashed — all lines update
6. **Change thickness** to 3 — all lines update
7. **Daily chart, Daily timeframe** — no lines rendered (guard: chart TF not lower)

- [ ] **Step 3: Commit**

```bash
git add prev-htf-high-low.pine
git commit -m "feat: finalize Previous HTF High/Low indicator"
```
