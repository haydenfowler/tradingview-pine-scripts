# Pine Scripts

A collection of custom TradingView indicators and strategies written in Pine Script.

## Indicators

### HTF Candles
Displays higher timeframe candles overlaid on your current chart. Useful for seeing larger timeframe structure without switching charts.

**Features:**
- Auto-selects the next higher timeframe, or choose manually
- Customizable colors for bullish/bearish candles
- Adjustable line thickness and style
- Optional midline per candle — `(high + low) / 2` of the HTF range, in the candle's own bull/bear colour
- Projects incomplete HTF candles to their expected width

Note: TradingView caps an indicator at 500 lines. Each candle uses two (the wicks), or three with
the midline on, so the oldest candles lose their wicks and midline sooner when it is enabled. Their
bodies are boxes and are unaffected.

### ATR + Candle Movement
Plots ATR alongside per-candle movement to visualize volatility relative to the average.

**Features:**
- Choose between High-Low range or True Range for candle movement
- Multiple smoothing options (RMA, SMA, EMA, WMA)
- Optional candle/ATR ratio display
- Option to show values as percentage of price

### Previous Day OHLCM
Displays the prior day's Open, High, Low, Close, and Midpoint as step lines on your chart.

**Features:**
- Shows all five prior-day levels (O, H, L, C, Mid) as persistent reference lines
- Step line style for clean horizontal level display
- Toggle visibility on/off

### Previous Day Midpoint
Draws the prior day's midpoint — `(high + low) / 2` — as a labelled horizontal segment across each day, on any daily or intraday chart.

**Features:**
- One non-overlapping segment per day, each showing the previous day's midpoint
- Today's segment is projected forward to the session close, ahead of price
- Labels are relative to today (`D-1 M`, `D-2 M`, …) and renumber as days pass
- Configurable number of days to display
- Configurable line colour, width, and style (solid/dotted/dashed)
- Configurable label visibility, position, colour, and size

### Previous HTF High/Low
Plots the high and low of the previous completed bar on a higher timeframe as horizontal lines.

**Features:**
- Auto-selects the next higher timeframe, or choose manually
- Configurable line color, thickness, and style (solid/dotted/dashed)

### Simple Sessions
Highlights global trading sessions (US, EU, Asia/Tokyo, and a user-defined session) as background fills on the chart.

**Features:**
- Toggle each session on/off with customizable fill colors
- Accounts for daylight saving time across regions
- Shows session high/low lines
- Optional user-defined custom session with timezone support

## Installation

1. Open TradingView and go to the Pine Script Editor
2. Copy the contents of the desired `.pine` file
3. Paste into the editor and click "Add to Chart"

## License

[Mozilla Public License 2.0](https://mozilla.org/MPL/2.0/)
