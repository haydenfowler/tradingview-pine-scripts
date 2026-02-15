# Pine Scripts

A collection of custom TradingView indicators and strategies written in Pine Script.

## Indicators

### HTF Candles
Displays higher timeframe candles overlaid on your current chart. Useful for seeing larger timeframe structure without switching charts.

**Features:**
- Auto-selects the next higher timeframe, or choose manually
- Customizable colors for bullish/bearish candles
- Adjustable line thickness and style
- Projects incomplete HTF candles to their expected width

### ATR + Candle Movement
Plots ATR alongside per-candle movement to visualize volatility relative to the average.

**Features:**
- Choose between High-Low range or True Range for candle movement
- Multiple smoothing options (RMA, SMA, EMA, WMA)
- Optional candle/ATR ratio display
- Option to show values as percentage of price

## Installation

1. Open TradingView and go to the Pine Script Editor
2. Copy the contents of the desired `.pine` file
3. Paste into the editor and click "Add to Chart"

## License

[Mozilla Public License 2.0](https://mozilla.org/MPL/2.0/)
