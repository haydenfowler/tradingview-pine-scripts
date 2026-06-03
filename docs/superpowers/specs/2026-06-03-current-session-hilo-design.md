# Current Session High/Low Lines — Design Spec

**Date:** 2026-06-03
**Script:** `simple-sessions.pine`

## Overview

Add a toggle to display live high/low lines for the session currently in progress. Lines update each bar as price extends the session H/L and are removed when the session closes (handing off to the existing closed-session H/L lines).

## New Input

```pine
showCurrentSessHiLo = input.bool(
    title   = "Show Current Session High/Low",
    defval  = true,
    group   = "Session High/Low Settings",
    tooltip = "Draws live high/low lines for the session currently in progress.",
    display = display.none
)
```

Placed in the existing "Session High/Low Settings" group, alongside `showSessionHiLo`. The two toggles are independent — users can show live H/L without historical lines, or vice versa.

## Per-Session State

Two new `var line` variables per session to hold the live lines:

| Session | Variables |
|---------|-----------|
| JP      | `jpCurrHighLine`, `jpCurrLowLine` |
| EU      | `euCurrHighLine`, `euCurrLowLine` |
| US      | `usCurrHighLine`, `usCurrLowLine` |
| Custom  | `custCurrHighLine`, `custCurrLowLine` |

Initialised to `na`. No arrays needed — only one current session instance exists at a time.

## Line Lifecycle

### Session start (`isXSession and not xWasInSess`)

If `showCurrentSessHiLo and showXSession`:
- Delete any stale lines (guard against non-`na` from a prior run)
- Create `xCurrHighLine` and `xCurrLowLine` from `bar_index` to `bar_index`, using `xSessHigh`/`xSessLow`, the session's line color, `lineStyleVal`, and `hiLoLineWidth`

### Each bar during session (`isXSession`)

If `showCurrentSessHiLo and showXSession` and lines are not `na`:
- `line.set_x2(xCurrHighLine, bar_index)` and `line.set_y1/y2` to `xSessHigh`
- Same for low line with `xSessLow`

(H/L tracking updates `xSessHigh`/`xSessLow` before the line update block runs, so lines always reflect the current bar's state.)

### Session end (`not isXSession and xWasInSess`)

Before drawing closed-session lines:
- `line.delete(xCurrHighLine)` / `line.delete(xCurrLowLine)`
- Set both to `na`

This ensures no visual overlap with the persistent closed-session lines drawn immediately after.

## Styling

Live lines use the same style as closed-session lines: `lineStyleVal` (Solid/Dashed/Dotted), `hiLoLineWidth`, and the session's `xLineColor`. No distinct "in-progress" styling — consistency is preferred.

## Scope

Applies to all four sessions: JP, EU, US, Custom. Each respects its own `showXSession` gate in addition to the global `showCurrentSessHiLo` toggle.

The feature does not apply on non-intraday timeframes (guarded by the existing `isInSession` function which returns `false` on daily+).
