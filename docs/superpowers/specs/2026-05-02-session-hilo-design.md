# Session High/Low Lines — Design Spec

## Overview

Extend `simple-sessions.pine` to display dotted high/low lines for each closed session, in the session's colour, extending to the right edge of the chart. During an active session, show a dotted line at the session open price instead.

## New Inputs

| Input | Type | Default | Notes |
|---|---|---|---|
| `showSessionHiLo` | bool | true | Global toggle — one setting controls all sessions |
| `hiLoLookback` | int | 1 | Number of past closed sessions to display (range 1–10) |

These sit in their own input group: `"Session High/Low Settings"`.

## Colour Changes

Update the three session background colour defaults:

| Session | New default colour |
|---|---|
| JP / Asia (Tokyo) | Lightest yellow, 15% opacity — `color.rgb(255, 255, 204, 85)` |
| EU / London | Lightest blue, 15% opacity — `color.rgb(204, 229, 255, 85)` |
| US / New York | Lightest purple, 15% opacity — `color.rgb(229, 204, 255, 85)` |

The AU (hidden) and Mixed sessions are unchanged.

## Per-Session State (applied to US, EU, JP, Custom)

For each session, declare the following `var` state:

```
var float  sessHigh      = na   // running high for current session
var float  sessLow       = na   // running low for current session
var float  sessOpen      = na   // first bar's open for current session
var bool   wasInSess     = false
var line   openLine      = na   // dotted open-price line while session active
var line[] highLines     = array.new_line()  // most recent N high lines
var line[] lowLines      = array.new_line()  // most recent N low lines
```

## Bar-by-Bar Logic

Each bar, for each session:

1. **Session start** (`isSession and not wasInSess`):
   - Record `sessOpen = open`, reset `sessHigh = high`, `sessLow = low`
   - Delete any existing `openLine`
   - Draw new dotted `openLine` at `sessOpen`, `extend.right`, session line colour

2. **Mid-session** (`isSession`):
   - `sessHigh := math.max(sessHigh, high)`
   - `sessLow  := math.min(sessLow,  low)`

3. **Session end** (`not isSession and wasInSess`):
   - Delete `openLine`
   - Draw dotted high line at `sessHigh`, `extend.right`, session line colour
   - Draw dotted low line  at `sessLow`,  `extend.right`, session line colour
   - Push both onto their arrays; if `array.size > hiLoLookback`, delete the oldest line and remove from array

4. Update `wasInSess := isSession`

All drawing is gated on `showSessionHiLo and showXxxSession`.

## Line Style

| Property | Value |
|---|---|
| Style | `line.style_dotted` |
| Width | 1 |
| Extend | `extend.right` |
| x1 | `bar_index` at session start |
| x2 | `bar_index` at session end (line still extends right via `extend.right`) |
| Colour | Session bg colour passed through `color.new(sessColor, 30)` (70% opacity) |

Using `color.new(sessColor, 30)` strips the heavy transparency from the bg colour so the line is clearly visible while still matching the session's hue.

## Indicator Header Change

Update `max_lines_count` to accommodate up to 10 sessions × 4 sessions × 2 lines = 80, plus open lines. Set `max_lines_count = 200` to give comfortable headroom.

## Out of Scope

- Mixed (EU/US overlap) session: no H/L lines
- Line labels
- Per-session H/L toggles (one global toggle covers all)
