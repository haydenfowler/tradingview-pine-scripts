# Current Session High/Low Lines — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `showCurrentSessHiLo` toggle (enabled by default) that draws live high/low lines for the session currently in progress, updating each bar and removing them when the session closes.

**Architecture:** One new global bool input in "Session High/Low Settings". Each of the four sessions (JP, EU, US, Custom) gets two new `var line` state variables for the live H/L lines. Lines are created on session start, updated each bar, and deleted on session end before the closed-session lines are drawn — no overlap.

**Tech Stack:** Pine Script v6, TradingView indicator. No test framework — verification is done by loading the script in TradingView and inspecting behaviour on an intraday chart.

---

### Task 1: Add the `showCurrentSessHiLo` input

**Files:**
- Modify: `simple-sessions.pine:39-44` (Session High/Low Settings input block)

- [ ] **Step 1: Add the input after `showSessionHiLo`**

In `simple-sessions.pine`, find this block (lines 39–44):

```pine
showSessionHiLo = input.bool(title="Show Session High/Low Lines", defval=true, group="Session High/Low Settings",
     tooltip="Draws lines from the candle that made the session high/low, extending right until price invalidates that level.", display=display.none)
hiLoLookback    = input.int(title="Sessions to Display", defval=1, minval=1, maxval=10, group="Session High/Low Settings",
     tooltip="Number of past closed sessions for which to display high/low lines.", display=display.none)
hiLoLineWidth   = input.int(title="Line Width", defval=1, minval=1, maxval=4, group="Session High/Low Settings", display=display.none)
hiLoLineStyle   = input.string(title="Line Style", defval="Solid", options=["Solid", "Dashed", "Dotted"], group="Session High/Low Settings", display=display.none)
```

Replace with:

```pine
showSessionHiLo     = input.bool(title="Show Session High/Low Lines", defval=true, group="Session High/Low Settings",
     tooltip="Draws lines from the candle that made the session high/low, extending right until price invalidates that level.", display=display.none)
showCurrentSessHiLo = input.bool(title="Show Current Session High/Low", defval=true, group="Session High/Low Settings",
     tooltip="Draws live high/low lines for the session currently in progress. Lines are removed when the session closes.", display=display.none)
hiLoLookback        = input.int(title="Sessions to Display", defval=1, minval=1, maxval=10, group="Session High/Low Settings",
     tooltip="Number of past closed sessions for which to display high/low lines.", display=display.none)
hiLoLineWidth       = input.int(title="Line Width", defval=1, minval=1, maxval=4, group="Session High/Low Settings", display=display.none)
hiLoLineStyle       = input.string(title="Line Style", defval="Solid", options=["Solid", "Dashed", "Dotted"], group="Session High/Low Settings", display=display.none)
```

- [ ] **Step 2: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: add showCurrentSessHiLo input to session H/L settings"
```

---

### Task 2: JP session — live current H/L lines

**Files:**
- Modify: `simple-sessions.pine` — JP session block (lines 82–168)

- [ ] **Step 1: Add state variables for the live JP lines**

Find the JP state variable declarations (around line 83):

```pine
// ─── JP/Asia Session High/Low Lines ───────────────────────────────────────────
var float   jpSessHigh    = na
var float   jpSessLow     = na
var float   jpSessOpen    = na
var int     jpSessHighBar = na
var int     jpSessLowBar  = na
var bool    jpWasInSess   = false
var line    jpOpenLine    = na
var line[]  jpHighLines   = array.new_line()
```

Replace with:

```pine
// ─── JP/Asia Session High/Low Lines ───────────────────────────────────────────
var float   jpSessHigh     = na
var float   jpSessLow      = na
var float   jpSessOpen     = na
var int     jpSessHighBar  = na
var int     jpSessLowBar   = na
var bool    jpWasInSess    = false
var line    jpOpenLine     = na
var line    jpCurrHighLine = na
var line    jpCurrLowLine  = na
var line[]  jpHighLines    = array.new_line()
```

- [ ] **Step 2: Create live lines on JP session start**

Find the JP session-start block:

```pine
if isJPSession and not jpWasInSess
    if showSessionHiLo and showJPSession and array.size(jpHighLines) > 0
        for i = 0 to array.size(jpHighLines) - 1
            if not array.get(jpHighDone, i)
                line.set_x2(array.get(jpHighLines, i), bar_index)
                array.set(jpHighDone, i, true)
        for i = 0 to array.size(jpLowLines) - 1
            if not array.get(jpLowDone, i)
                line.set_x2(array.get(jpLowLines, i), bar_index)
                array.set(jpLowDone, i, true)
    jpSessHighBar := bar_index
    jpSessLowBar  := bar_index
    jpSessOpen    := open
    jpSessHigh    := high
    jpSessLow     := low
    if showSessionHiLo and showJPSession
        if not na(jpOpenLine)
            line.delete(jpOpenLine)
        jpOpenLine := line.new(bar_index, jpSessOpen, bar_index, jpSessOpen,
                               color=jpLineColor, style=line.style_dashed,
                               width=hiLoLineWidth, extend=extend.right)
```

Replace with:

```pine
if isJPSession and not jpWasInSess
    if showSessionHiLo and showJPSession and array.size(jpHighLines) > 0
        for i = 0 to array.size(jpHighLines) - 1
            if not array.get(jpHighDone, i)
                line.set_x2(array.get(jpHighLines, i), bar_index)
                array.set(jpHighDone, i, true)
        for i = 0 to array.size(jpLowLines) - 1
            if not array.get(jpLowDone, i)
                line.set_x2(array.get(jpLowLines, i), bar_index)
                array.set(jpLowDone, i, true)
    jpSessHighBar := bar_index
    jpSessLowBar  := bar_index
    jpSessOpen    := open
    jpSessHigh    := high
    jpSessLow     := low
    if showSessionHiLo and showJPSession
        if not na(jpOpenLine)
            line.delete(jpOpenLine)
        jpOpenLine := line.new(bar_index, jpSessOpen, bar_index, jpSessOpen,
                               color=jpLineColor, style=line.style_dashed,
                               width=hiLoLineWidth, extend=extend.right)
    if showCurrentSessHiLo and showJPSession
        if not na(jpCurrHighLine)
            line.delete(jpCurrHighLine)
        if not na(jpCurrLowLine)
            line.delete(jpCurrLowLine)
        jpCurrHighLine := line.new(bar_index, jpSessHigh, bar_index, jpSessHigh,
                                   color=jpLineColor, style=lineStyleVal, width=hiLoLineWidth)
        jpCurrLowLine  := line.new(bar_index, jpSessLow,  bar_index, jpSessLow,
                                   color=jpLineColor, style=lineStyleVal, width=hiLoLineWidth)
```

- [ ] **Step 3: Update live lines each bar during JP session**

Find the in-session H/L tracking block:

```pine
if isJPSession
    if high > jpSessHigh
        jpSessHigh    := high
        jpSessHighBar := bar_index
    if low < jpSessLow
        jpSessLow    := low
        jpSessLowBar := bar_index
```

Replace with:

```pine
if isJPSession
    if high > jpSessHigh
        jpSessHigh    := high
        jpSessHighBar := bar_index
    if low < jpSessLow
        jpSessLow    := low
        jpSessLowBar := bar_index
    if showCurrentSessHiLo and showJPSession
        if not na(jpCurrHighLine)
            line.set_x2(jpCurrHighLine, bar_index)
            line.set_y1(jpCurrHighLine, jpSessHigh)
            line.set_y2(jpCurrHighLine, jpSessHigh)
        if not na(jpCurrLowLine)
            line.set_x2(jpCurrLowLine, bar_index)
            line.set_y1(jpCurrLowLine, jpSessLow)
            line.set_y2(jpCurrLowLine, jpSessLow)
```

- [ ] **Step 4: Delete live lines on JP session end**

Find the session-end block:

```pine
if not isJPSession and jpWasInSess
    if showSessionHiLo and showJPSession
        if not na(jpOpenLine)
            line.delete(jpOpenLine)
            jpOpenLine := na
        _h = line.new(jpSessHighBar, jpSessHigh, bar_index, jpSessHigh,
```

Replace with:

```pine
if not isJPSession and jpWasInSess
    if showCurrentSessHiLo and showJPSession
        line.delete(jpCurrHighLine)
        line.delete(jpCurrLowLine)
        jpCurrHighLine := na
        jpCurrLowLine  := na
    if showSessionHiLo and showJPSession
        if not na(jpOpenLine)
            line.delete(jpOpenLine)
            jpOpenLine := na
        _h = line.new(jpSessHighBar, jpSessHigh, bar_index, jpSessHigh,
```

- [ ] **Step 5: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: add live current H/L lines for JP session"
```

---

### Task 3: EU session — live current H/L lines

**Files:**
- Modify: `simple-sessions.pine` — EU session block (lines 170–256)

- [ ] **Step 1: Add state variables for the live EU lines**

Find the EU state variable declarations:

```pine
// ─── EU/London Session High/Low Lines ─────────────────────────────────────────
var float   euSessHigh    = na
var float   euSessLow     = na
var float   euSessOpen    = na
var int     euSessHighBar = na
var int     euSessLowBar  = na
var bool    euWasInSess   = false
var line    euOpenLine    = na
var line[]  euHighLines   = array.new_line()
```

Replace with:

```pine
// ─── EU/London Session High/Low Lines ─────────────────────────────────────────
var float   euSessHigh     = na
var float   euSessLow      = na
var float   euSessOpen     = na
var int     euSessHighBar  = na
var int     euSessLowBar   = na
var bool    euWasInSess    = false
var line    euOpenLine     = na
var line    euCurrHighLine = na
var line    euCurrLowLine  = na
var line[]  euHighLines    = array.new_line()
```

- [ ] **Step 2: Create live lines on EU session start**

Find the EU session-start block:

```pine
if isEUSession and not euWasInSess
    if showSessionHiLo and showEUSession and array.size(euHighLines) > 0
        for i = 0 to array.size(euHighLines) - 1
            if not array.get(euHighDone, i)
                line.set_x2(array.get(euHighLines, i), bar_index)
                array.set(euHighDone, i, true)
        for i = 0 to array.size(euLowLines) - 1
            if not array.get(euLowDone, i)
                line.set_x2(array.get(euLowLines, i), bar_index)
                array.set(euLowDone, i, true)
    euSessHighBar := bar_index
    euSessLowBar  := bar_index
    euSessOpen    := open
    euSessHigh    := high
    euSessLow     := low
    if showSessionHiLo and showEUSession
        if not na(euOpenLine)
            line.delete(euOpenLine)
        euOpenLine := line.new(bar_index, euSessOpen, bar_index, euSessOpen,
                               color=euLineColor, style=line.style_dashed,
                               width=hiLoLineWidth, extend=extend.right)
```

Replace with:

```pine
if isEUSession and not euWasInSess
    if showSessionHiLo and showEUSession and array.size(euHighLines) > 0
        for i = 0 to array.size(euHighLines) - 1
            if not array.get(euHighDone, i)
                line.set_x2(array.get(euHighLines, i), bar_index)
                array.set(euHighDone, i, true)
        for i = 0 to array.size(euLowLines) - 1
            if not array.get(euLowDone, i)
                line.set_x2(array.get(euLowLines, i), bar_index)
                array.set(euLowDone, i, true)
    euSessHighBar := bar_index
    euSessLowBar  := bar_index
    euSessOpen    := open
    euSessHigh    := high
    euSessLow     := low
    if showSessionHiLo and showEUSession
        if not na(euOpenLine)
            line.delete(euOpenLine)
        euOpenLine := line.new(bar_index, euSessOpen, bar_index, euSessOpen,
                               color=euLineColor, style=line.style_dashed,
                               width=hiLoLineWidth, extend=extend.right)
    if showCurrentSessHiLo and showEUSession
        if not na(euCurrHighLine)
            line.delete(euCurrHighLine)
        if not na(euCurrLowLine)
            line.delete(euCurrLowLine)
        euCurrHighLine := line.new(bar_index, euSessHigh, bar_index, euSessHigh,
                                   color=euLineColor, style=lineStyleVal, width=hiLoLineWidth)
        euCurrLowLine  := line.new(bar_index, euSessLow,  bar_index, euSessLow,
                                   color=euLineColor, style=lineStyleVal, width=hiLoLineWidth)
```

- [ ] **Step 3: Update live lines each bar during EU session**

Find:

```pine
if isEUSession
    if high > euSessHigh
        euSessHigh    := high
        euSessHighBar := bar_index
    if low < euSessLow
        euSessLow    := low
        euSessLowBar := bar_index
```

Replace with:

```pine
if isEUSession
    if high > euSessHigh
        euSessHigh    := high
        euSessHighBar := bar_index
    if low < euSessLow
        euSessLow    := low
        euSessLowBar := bar_index
    if showCurrentSessHiLo and showEUSession
        if not na(euCurrHighLine)
            line.set_x2(euCurrHighLine, bar_index)
            line.set_y1(euCurrHighLine, euSessHigh)
            line.set_y2(euCurrHighLine, euSessHigh)
        if not na(euCurrLowLine)
            line.set_x2(euCurrLowLine, bar_index)
            line.set_y1(euCurrLowLine, euSessLow)
            line.set_y2(euCurrLowLine, euSessLow)
```

- [ ] **Step 4: Delete live lines on EU session end**

Find:

```pine
if not isEUSession and euWasInSess
    if showSessionHiLo and showEUSession
        if not na(euOpenLine)
            line.delete(euOpenLine)
            euOpenLine := na
        _h = line.new(euSessHighBar, euSessHigh, bar_index, euSessHigh,
```

Replace with:

```pine
if not isEUSession and euWasInSess
    if showCurrentSessHiLo and showEUSession
        line.delete(euCurrHighLine)
        line.delete(euCurrLowLine)
        euCurrHighLine := na
        euCurrLowLine  := na
    if showSessionHiLo and showEUSession
        if not na(euOpenLine)
            line.delete(euOpenLine)
            euOpenLine := na
        _h = line.new(euSessHighBar, euSessHigh, bar_index, euSessHigh,
```

- [ ] **Step 5: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: add live current H/L lines for EU session"
```

---

### Task 4: US session — live current H/L lines

**Files:**
- Modify: `simple-sessions.pine` — US session block (lines 258–344)

- [ ] **Step 1: Add state variables for the live US lines**

Find:

```pine
// ─── US/NY Session High/Low Lines ─────────────────────────────────────────────
var float   usSessHigh    = na
var float   usSessLow     = na
var float   usSessOpen    = na
var int     usSessHighBar = na
var int     usSessLowBar  = na
var bool    usWasInSess   = false
var line    usOpenLine    = na
var line[]  usHighLines   = array.new_line()
```

Replace with:

```pine
// ─── US/NY Session High/Low Lines ─────────────────────────────────────────────
var float   usSessHigh     = na
var float   usSessLow      = na
var float   usSessOpen     = na
var int     usSessHighBar  = na
var int     usSessLowBar   = na
var bool    usWasInSess    = false
var line    usOpenLine     = na
var line    usCurrHighLine = na
var line    usCurrLowLine  = na
var line[]  usHighLines    = array.new_line()
```

- [ ] **Step 2: Create live lines on US session start**

Find:

```pine
if isUSSession and not usWasInSess
    if showSessionHiLo and showUSSession and array.size(usHighLines) > 0
        for i = 0 to array.size(usHighLines) - 1
            if not array.get(usHighDone, i)
                line.set_x2(array.get(usHighLines, i), bar_index)
                array.set(usHighDone, i, true)
        for i = 0 to array.size(usLowLines) - 1
            if not array.get(usLowDone, i)
                line.set_x2(array.get(usLowLines, i), bar_index)
                array.set(usLowDone, i, true)
    usSessHighBar := bar_index
    usSessLowBar  := bar_index
    usSessOpen    := open
    usSessHigh    := high
    usSessLow     := low
    if showSessionHiLo and showUSSession
        if not na(usOpenLine)
            line.delete(usOpenLine)
        usOpenLine := line.new(bar_index, usSessOpen, bar_index, usSessOpen,
                               color=usLineColor, style=line.style_dashed,
                               width=hiLoLineWidth, extend=extend.right)
```

Replace with:

```pine
if isUSSession and not usWasInSess
    if showSessionHiLo and showUSSession and array.size(usHighLines) > 0
        for i = 0 to array.size(usHighLines) - 1
            if not array.get(usHighDone, i)
                line.set_x2(array.get(usHighLines, i), bar_index)
                array.set(usHighDone, i, true)
        for i = 0 to array.size(usLowLines) - 1
            if not array.get(usLowDone, i)
                line.set_x2(array.get(usLowLines, i), bar_index)
                array.set(usLowDone, i, true)
    usSessHighBar := bar_index
    usSessLowBar  := bar_index
    usSessOpen    := open
    usSessHigh    := high
    usSessLow     := low
    if showSessionHiLo and showUSSession
        if not na(usOpenLine)
            line.delete(usOpenLine)
        usOpenLine := line.new(bar_index, usSessOpen, bar_index, usSessOpen,
                               color=usLineColor, style=line.style_dashed,
                               width=hiLoLineWidth, extend=extend.right)
    if showCurrentSessHiLo and showUSSession
        if not na(usCurrHighLine)
            line.delete(usCurrHighLine)
        if not na(usCurrLowLine)
            line.delete(usCurrLowLine)
        usCurrHighLine := line.new(bar_index, usSessHigh, bar_index, usSessHigh,
                                   color=usLineColor, style=lineStyleVal, width=hiLoLineWidth)
        usCurrLowLine  := line.new(bar_index, usSessLow,  bar_index, usSessLow,
                                   color=usLineColor, style=lineStyleVal, width=hiLoLineWidth)
```

- [ ] **Step 3: Update live lines each bar during US session**

Find:

```pine
if isUSSession
    if high > usSessHigh
        usSessHigh    := high
        usSessHighBar := bar_index
    if low < usSessLow
        usSessLow    := low
        usSessLowBar := bar_index
```

Replace with:

```pine
if isUSSession
    if high > usSessHigh
        usSessHigh    := high
        usSessHighBar := bar_index
    if low < usSessLow
        usSessLow    := low
        usSessLowBar := bar_index
    if showCurrentSessHiLo and showUSSession
        if not na(usCurrHighLine)
            line.set_x2(usCurrHighLine, bar_index)
            line.set_y1(usCurrHighLine, usSessHigh)
            line.set_y2(usCurrHighLine, usSessHigh)
        if not na(usCurrLowLine)
            line.set_x2(usCurrLowLine, bar_index)
            line.set_y1(usCurrLowLine, usSessLow)
            line.set_y2(usCurrLowLine, usSessLow)
```

- [ ] **Step 4: Delete live lines on US session end**

Find:

```pine
if not isUSSession and usWasInSess
    if showSessionHiLo and showUSSession
        if not na(usOpenLine)
            line.delete(usOpenLine)
            usOpenLine := na
        _h = line.new(usSessHighBar, usSessHigh, bar_index, usSessHigh,
```

Replace with:

```pine
if not isUSSession and usWasInSess
    if showCurrentSessHiLo and showUSSession
        line.delete(usCurrHighLine)
        line.delete(usCurrLowLine)
        usCurrHighLine := na
        usCurrLowLine  := na
    if showSessionHiLo and showUSSession
        if not na(usOpenLine)
            line.delete(usOpenLine)
            usOpenLine := na
        _h = line.new(usSessHighBar, usSessHigh, bar_index, usSessHigh,
```

- [ ] **Step 5: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: add live current H/L lines for US session"
```

---

### Task 5: Custom session — live current H/L lines

**Files:**
- Modify: `simple-sessions.pine` — Custom session block (lines 346–432)

- [ ] **Step 1: Add state variables for the live Custom lines**

Find:

```pine
// ─── Custom Session High/Low Lines ────────────────────────────────────────────
var float   custSessHigh    = na
var float   custSessLow     = na
var float   custSessOpen    = na
var int     custSessHighBar = na
var int     custSessLowBar  = na
var bool    custWasInSess   = false
var line    custOpenLine    = na
var line[]  custHighLines   = array.new_line()
```

Replace with:

```pine
// ─── Custom Session High/Low Lines ────────────────────────────────────────────
var float   custSessHigh     = na
var float   custSessLow      = na
var float   custSessOpen     = na
var int     custSessHighBar  = na
var int     custSessLowBar   = na
var bool    custWasInSess    = false
var line    custOpenLine     = na
var line    custCurrHighLine = na
var line    custCurrLowLine  = na
var line[]  custHighLines    = array.new_line()
```

- [ ] **Step 2: Create live lines on Custom session start**

Find:

```pine
if isCustomSession and not custWasInSess
    if showSessionHiLo and showCustomSession and array.size(custHighLines) > 0
        for i = 0 to array.size(custHighLines) - 1
            if not array.get(custHighDone, i)
                line.set_x2(array.get(custHighLines, i), bar_index)
                array.set(custHighDone, i, true)
        for i = 0 to array.size(custLowLines) - 1
            if not array.get(custLowDone, i)
                line.set_x2(array.get(custLowLines, i), bar_index)
                array.set(custLowDone, i, true)
    custSessHighBar := bar_index
    custSessLowBar  := bar_index
    custSessOpen    := open
    custSessHigh    := high
    custSessLow     := low
    if showSessionHiLo and showCustomSession
        if not na(custOpenLine)
            line.delete(custOpenLine)
        custOpenLine := line.new(bar_index, custSessOpen, bar_index, custSessOpen,
                                 color=custLineColor, style=line.style_dashed,
                                 width=hiLoLineWidth, extend=extend.right)
```

Replace with:

```pine
if isCustomSession and not custWasInSess
    if showSessionHiLo and showCustomSession and array.size(custHighLines) > 0
        for i = 0 to array.size(custHighLines) - 1
            if not array.get(custHighDone, i)
                line.set_x2(array.get(custHighLines, i), bar_index)
                array.set(custHighDone, i, true)
        for i = 0 to array.size(custLowLines) - 1
            if not array.get(custLowDone, i)
                line.set_x2(array.get(custLowLines, i), bar_index)
                array.set(custLowDone, i, true)
    custSessHighBar := bar_index
    custSessLowBar  := bar_index
    custSessOpen    := open
    custSessHigh    := high
    custSessLow     := low
    if showSessionHiLo and showCustomSession
        if not na(custOpenLine)
            line.delete(custOpenLine)
        custOpenLine := line.new(bar_index, custSessOpen, bar_index, custSessOpen,
                                 color=custLineColor, style=line.style_dashed,
                                 width=hiLoLineWidth, extend=extend.right)
    if showCurrentSessHiLo and showCustomSession
        if not na(custCurrHighLine)
            line.delete(custCurrHighLine)
        if not na(custCurrLowLine)
            line.delete(custCurrLowLine)
        custCurrHighLine := line.new(bar_index, custSessHigh, bar_index, custSessHigh,
                                     color=custLineColor, style=lineStyleVal, width=hiLoLineWidth)
        custCurrLowLine  := line.new(bar_index, custSessLow,  bar_index, custSessLow,
                                     color=custLineColor, style=lineStyleVal, width=hiLoLineWidth)
```

- [ ] **Step 3: Update live lines each bar during Custom session**

Find:

```pine
if isCustomSession
    if high > custSessHigh
        custSessHigh    := high
        custSessHighBar := bar_index
    if low < custSessLow
        custSessLow    := low
        custSessLowBar := bar_index
```

Replace with:

```pine
if isCustomSession
    if high > custSessHigh
        custSessHigh    := high
        custSessHighBar := bar_index
    if low < custSessLow
        custSessLow    := low
        custSessLowBar := bar_index
    if showCurrentSessHiLo and showCustomSession
        if not na(custCurrHighLine)
            line.set_x2(custCurrHighLine, bar_index)
            line.set_y1(custCurrHighLine, custSessHigh)
            line.set_y2(custCurrHighLine, custSessHigh)
        if not na(custCurrLowLine)
            line.set_x2(custCurrLowLine, bar_index)
            line.set_y1(custCurrLowLine, custSessLow)
            line.set_y2(custCurrLowLine, custSessLow)
```

- [ ] **Step 4: Delete live lines on Custom session end**

Find:

```pine
if not isCustomSession and custWasInSess
    if showSessionHiLo and showCustomSession
        if not na(custOpenLine)
            line.delete(custOpenLine)
            custOpenLine := na
        _h = line.new(custSessHighBar, custSessHigh, bar_index, custSessHigh,
```

Replace with:

```pine
if not isCustomSession and custWasInSess
    if showCurrentSessHiLo and showCustomSession
        line.delete(custCurrHighLine)
        line.delete(custCurrLowLine)
        custCurrHighLine := na
        custCurrLowLine  := na
    if showSessionHiLo and showCustomSession
        if not na(custOpenLine)
            line.delete(custOpenLine)
            custOpenLine := na
        _h = line.new(custSessHighBar, custSessHigh, bar_index, custSessHigh,
```

- [ ] **Step 5: Commit**

```bash
git add simple-sessions.pine
git commit -m "feat: add live current H/L lines for Custom session"
```

---

### Task 6: Verify in TradingView

- [ ] **Step 1: Load the script**

Open TradingView, go to Pine Script Editor, paste the updated `simple-sessions.pine` content, and click "Add to chart" on an intraday timeframe (e.g. 5m or 15m).

- [ ] **Step 2: Verify live H/L lines appear during an active session**

On a chart where at least one session (JP, EU, or US depending on current time) is active, confirm:
- A high line and a low line are drawn from the session open bar to the current bar
- They use the session's line color and the configured line style/width
- They extend and update as price makes new session highs/lows

- [ ] **Step 3: Verify live lines disappear at session close**

Wait for (or simulate by changing the chart timezone to) a session boundary. Confirm:
- The live H/L lines disappear exactly when the session ends
- The closed-session H/L lines (from `showSessionHiLo`) appear immediately at the same price levels — no overlap or gap

- [ ] **Step 4: Verify the toggle works**

In indicator settings → "Session High/Low Settings", uncheck "Show Current Session High/Low". Confirm the live lines disappear. Re-enable and confirm they return.

- [ ] **Step 5: Verify independence from `showSessionHiLo`**

Disable "Show Session High/Low Lines" but keep "Show Current Session High/Low" enabled. Confirm live lines still appear during active sessions and no closed-session lines are drawn.
