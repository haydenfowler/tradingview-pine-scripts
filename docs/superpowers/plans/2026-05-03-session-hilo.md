# Session High/Low Lines Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add dotted H/L lines per session to `simple-sessions.pine`, drawn at session close and extending right, with a session-open line shown during active sessions.

**Architecture:** All changes are in a single Pine Script file. State is tracked with `var` variables per session; line objects are stored in arrays so the oldest lines beyond the lookback limit can be deleted. Session transitions (start/end) are detected by comparing current and previous `wasInSession` booleans.

**Tech Stack:** Pine Script v5, TradingView chart editor.

---

## Files

- Modify: `simple-sessions.pine`

---

### Task 1: Update indicator header and session default colours

**Files:**
- Modify: `simple-sessions.pine`

- [ ] **Step 1: Update `indicator()` call to raise line limit**

In `simple-sessions.pine` line 25, change:
```pine
indicator(title='Simple Sessions - [TTF]', shorttitle='Simple Sessions', overlay=true, max_labels_count=500)
```
to:
```pine
indicator(title='Simple Sessions - [TTF]', shorttitle='Simple Sessions', overlay=true, max_labels_count=500, max_lines_count=200)
```

- [ ] **Step 2: Update US session default colour**

Line 69, change `defval=color.rgb(255, 150, 150,85)` to `defval=color.rgb(229, 204, 255, 85)`:
```pine
usSessionColor = input.color(title="", defval=color.rgb(229, 204, 255, 85), group="US Session Display Settings", inline="us1", tooltip="Check the box to " +
             "enable the session highlight for the US session.\n\n" +
             "If you wish to change the default session highlight color, you may do so here as well.")
```

- [ ] **Step 3: Update EU session default colour**

Line 75, change `defval=color.rgb(150, 255, 150,85)` to `defval=color.rgb(204, 229, 255, 85)`:
```pine
euSessionColor = input.color(title="", defval=color.rgb(204, 229, 255, 85), group="EU Session Display Settings", inline="eu", tooltip="Check the box to " +
             "enable the session highlight for the EU session.\n\n" +
             "If you wish to change the default session highlight color, you may do so here as well.")
```

- [ ] **Step 4: Update JP session default colour**

Line 97, change `defval=color.rgb(91, 156, 246, 75)` to `defval=color.rgb(255, 255, 204, 85)`:
```pine
jpSessionColor = input.color(title="", defval=color.rgb(255, 255, 204, 85), group="JP Session Display Settings", inline="jp", tooltip="Check the box to " +
             "enable the session highlight for the Tokyo/Asia/Japan session.\n\n" +
             "If you wish to change the default session highlight color, you may do so here as well.")
```

- [ ] **Step 5: Verify in TradingView**

Paste the file into the Pine Script editor and add to chart. Confirm:
- US session background is light purple
- EU session background is light blue
- JP session background is light yellow

- [ ] **Step 6: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: update session default colours and raise max_lines_count"
```

---

### Task 2: Add global H/L settings inputs

**Files:**
- Modify: `simple-sessions.pine`

- [ ] **Step 1: Add the two new inputs**

After line 116 (the `custSessTimezone` input block, before the `// Now that we have the custom session inputs` comment), insert:

```pine
//
// High/Low line settings
//
showSessionHiLo = input.bool(title="Show Session High/Low Lines", defval=true, group="Session High/Low Settings",
     tooltip="When enabled, draws dotted high and low lines for each closed session in the session's colour.\n\n" +
             "During an active session, a dotted line is shown at the session open price instead.")
hiLoLookback = input.int(title="Sessions to Display", defval=1, minval=1, maxval=10, group="Session High/Low Settings",
     tooltip="Number of past closed sessions for which to display high/low lines.")
```

- [ ] **Step 2: Verify in TradingView**

Add to chart. Open the indicator settings and confirm a new "Session High/Low Settings" group appears with the two inputs.

- [ ] **Step 3: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: add showSessionHiLo and hiLoLookback inputs"
```

---

### Task 3: JP/Asia session — H/L state and line drawing

**Files:**
- Modify: `simple-sessions.pine`

- [ ] **Step 1: Add JP H/L block at the end of the file**

Append the following after the final `bgcolor(...)` call (after line 260):

```pine
// ─── JP/Asia Session High/Low Lines ───────────────────────────────────────────
var float  jpSessHigh     = na
var float  jpSessLow      = na
var float  jpSessOpen     = na
var int    jpSessStartBar = na
var bool   jpWasInSess    = false
var line   jpOpenLine     = na
var line[] jpHighLines    = array.new_line()
var line[] jpLowLines     = array.new_line()

if isJPSession and not jpWasInSess
    jpSessStartBar := bar_index
    jpSessOpen     := open
    jpSessHigh     := high
    jpSessLow      := low
    if showSessionHiLo and showJPSession
        if not na(jpOpenLine)
            line.delete(jpOpenLine)
        jpOpenLine := line.new(bar_index, jpSessOpen, bar_index + 1, jpSessOpen,
                               color=color.new(jpSessionColor, 30), style=line.style_dotted,
                               width=1, extend=extend.right)

if isJPSession
    jpSessHigh := math.max(jpSessHigh, high)
    jpSessLow  := math.min(jpSessLow,  low)

if not isJPSession and jpWasInSess
    if showSessionHiLo and showJPSession
        if not na(jpOpenLine)
            line.delete(jpOpenLine)
            jpOpenLine := na
        _h = line.new(jpSessStartBar, jpSessHigh, bar_index, jpSessHigh,
                      color=color.new(jpSessionColor, 30), style=line.style_dotted,
                      width=1, extend=extend.right)
        _l = line.new(jpSessStartBar, jpSessLow, bar_index, jpSessLow,
                      color=color.new(jpSessionColor, 30), style=line.style_dotted,
                      width=1, extend=extend.right)
        array.push(jpHighLines, _h)
        array.push(jpLowLines,  _l)
        if array.size(jpHighLines) > hiLoLookback
            line.delete(array.shift(jpHighLines))
            line.delete(array.shift(jpLowLines))

jpWasInSess := isJPSession
```

- [ ] **Step 2: Verify in TradingView**

Add to chart on an instrument that has Asia session activity (e.g. USDJPY on a 15m or 1h timeframe). Enable JP session. Confirm:
- A dotted yellow line appears at the open price during the active JP session
- After the session closes, it is replaced by two dotted yellow lines at the session high and low
- The lines extend to the right edge of the chart
- With `hiLoLookback=1` only the most recent closed session shows lines

- [ ] **Step 3: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: add JP/Asia session high/low line tracking and drawing"
```

---

### Task 4: EU/London session — H/L state and line drawing

**Files:**
- Modify: `simple-sessions.pine`

- [ ] **Step 1: Add EU H/L block at the end of the file**

Append after the JP block added in Task 3:

```pine
// ─── EU/London Session High/Low Lines ─────────────────────────────────────────
var float  euSessHigh     = na
var float  euSessLow      = na
var float  euSessOpen     = na
var int    euSessStartBar = na
var bool   euWasInSess    = false
var line   euOpenLine     = na
var line[] euHighLines    = array.new_line()
var line[] euLowLines     = array.new_line()

if isEUSession and not euWasInSess
    euSessStartBar := bar_index
    euSessOpen     := open
    euSessHigh     := high
    euSessLow      := low
    if showSessionHiLo and showEUSession
        if not na(euOpenLine)
            line.delete(euOpenLine)
        euOpenLine := line.new(bar_index, euSessOpen, bar_index + 1, euSessOpen,
                               color=color.new(euSessionColor, 30), style=line.style_dotted,
                               width=1, extend=extend.right)

if isEUSession
    euSessHigh := math.max(euSessHigh, high)
    euSessLow  := math.min(euSessLow,  low)

if not isEUSession and euWasInSess
    if showSessionHiLo and showEUSession
        if not na(euOpenLine)
            line.delete(euOpenLine)
            euOpenLine := na
        _h = line.new(euSessStartBar, euSessHigh, bar_index, euSessHigh,
                      color=color.new(euSessionColor, 30), style=line.style_dotted,
                      width=1, extend=extend.right)
        _l = line.new(euSessStartBar, euSessLow, bar_index, euSessLow,
                      color=color.new(euSessionColor, 30), style=line.style_dotted,
                      width=1, extend=extend.right)
        array.push(euHighLines, _h)
        array.push(euLowLines,  _l)
        if array.size(euHighLines) > hiLoLookback
            line.delete(array.shift(euHighLines))
            line.delete(array.shift(euLowLines))

euWasInSess := isEUSession
```

- [ ] **Step 2: Verify in TradingView**

Add to chart on e.g. EURUSD 1h. Enable EU session. Confirm:
- Dotted blue open line appears at session start
- Two dotted blue H/L lines appear after session close, extending right
- JP lines still work correctly alongside EU lines

- [ ] **Step 3: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: add EU/London session high/low line tracking and drawing"
```

---

### Task 5: US/NY session — H/L state and line drawing

**Files:**
- Modify: `simple-sessions.pine`

- [ ] **Step 1: Add US H/L block at the end of the file**

Append after the EU block added in Task 4:

```pine
// ─── US/NY Session High/Low Lines ─────────────────────────────────────────────
var float  usSessHigh     = na
var float  usSessLow      = na
var float  usSessOpen     = na
var int    usSessStartBar = na
var bool   usWasInSess    = false
var line   usOpenLine     = na
var line[] usHighLines    = array.new_line()
var line[] usLowLines     = array.new_line()

if isUSSession and not usWasInSess
    usSessStartBar := bar_index
    usSessOpen     := open
    usSessHigh     := high
    usSessLow      := low
    if showSessionHiLo and showUSSession
        if not na(usOpenLine)
            line.delete(usOpenLine)
        usOpenLine := line.new(bar_index, usSessOpen, bar_index + 1, usSessOpen,
                               color=color.new(usSessionColor, 30), style=line.style_dotted,
                               width=1, extend=extend.right)

if isUSSession
    usSessHigh := math.max(usSessHigh, high)
    usSessLow  := math.min(usSessLow,  low)

if not isUSSession and usWasInSess
    if showSessionHiLo and showUSSession
        if not na(usOpenLine)
            line.delete(usOpenLine)
            usOpenLine := na
        _h = line.new(usSessStartBar, usSessHigh, bar_index, usSessHigh,
                      color=color.new(usSessionColor, 30), style=line.style_dotted,
                      width=1, extend=extend.right)
        _l = line.new(usSessStartBar, usSessLow, bar_index, usSessLow,
                      color=color.new(usSessionColor, 30), style=line.style_dotted,
                      width=1, extend=extend.right)
        array.push(usHighLines, _h)
        array.push(usLowLines,  _l)
        if array.size(usHighLines) > hiLoLookback
            line.delete(array.shift(usHighLines))
            line.delete(array.shift(usLowLines))

usWasInSess := isUSSession
```

- [ ] **Step 2: Verify in TradingView**

Add to chart on e.g. SPY or EURUSD 1h. Enable US session. Confirm:
- Dotted purple open line at session start
- Two dotted purple H/L lines after session close
- All three sessions (JP, EU, US) work simultaneously without interfering

- [ ] **Step 3: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: add US/NY session high/low line tracking and drawing"
```

---

### Task 6: Custom session — H/L state and line drawing

**Files:**
- Modify: `simple-sessions.pine`

- [ ] **Step 1: Add Custom H/L block at the end of the file**

Append after the US block added in Task 5:

```pine
// ─── Custom Session High/Low Lines ────────────────────────────────────────────
var float  custSessHigh     = na
var float  custSessLow      = na
var float  custSessOpen     = na
var int    custSessStartBar = na
var bool   custWasInSess    = false
var line   custOpenLine     = na
var line[] custHighLines    = array.new_line()
var line[] custLowLines     = array.new_line()

if isCustomSession and not custWasInSess
    custSessStartBar := bar_index
    custSessOpen     := open
    custSessHigh     := high
    custSessLow      := low
    if showSessionHiLo and showCustomSession
        if not na(custOpenLine)
            line.delete(custOpenLine)
        custOpenLine := line.new(bar_index, custSessOpen, bar_index + 1, custSessOpen,
                                 color=color.new(custSessionColor, 30), style=line.style_dotted,
                                 width=1, extend=extend.right)

if isCustomSession
    custSessHigh := math.max(custSessHigh, high)
    custSessLow  := math.min(custSessLow,  low)

if not isCustomSession and custWasInSess
    if showSessionHiLo and showCustomSession
        if not na(custOpenLine)
            line.delete(custOpenLine)
            custOpenLine := na
        _h = line.new(custSessStartBar, custSessHigh, bar_index, custSessHigh,
                      color=color.new(custSessionColor, 30), style=line.style_dotted,
                      width=1, extend=extend.right)
        _l = line.new(custSessStartBar, custSessLow, bar_index, custSessLow,
                      color=color.new(custSessionColor, 30), style=line.style_dotted,
                      width=1, extend=extend.right)
        array.push(custHighLines, _h)
        array.push(custLowLines,  _l)
        if array.size(custHighLines) > hiLoLookback
            line.delete(array.shift(custHighLines))
            line.delete(array.shift(custLowLines))

custWasInSess := isCustomSession
```

- [ ] **Step 2: Verify in TradingView**

Enable the custom session with a time window that includes recent bars. Confirm:
- Dotted grey open line appears during the custom session
- H/L lines replace it after the session closes
- Toggling `showCustomSession` off hides the custom H/L lines without affecting the other sessions

- [ ] **Step 3: End-to-end checks**

With all 4 sessions enabled and `hiLoLookback=3`, confirm:
- Three past sessions worth of H/L lines appear for each session
- The 4th-oldest set of lines is automatically removed as new sessions close
- Toggling `showSessionHiLo` off hides all H/L and open lines globally

- [ ] **Step 4: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: add custom session high/low line tracking and drawing"
```
