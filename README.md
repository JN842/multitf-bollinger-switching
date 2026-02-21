# multitf-bollinger-switching

> Multi-timeframe Bollinger Band switching strategy for crypto backtesting — Python & Pine Script

A Bollinger Band **regime-switching** strategy that trades **ETHUSDT perpetual** using 1-minute execution logic, while using a 15-minute higher timeframe to classify the market as either a **breakout** or **mean-reversion** regime.

## Backtest Summary

| Parameter | Value |
|---|---|
| Instrument | ETHUSDT Perpetual (linear USDT-margined) |
| Data | ETHUSDT Spot 1m OHLCV (spot as perp proxy) |
| Period | 2024-02-10 → 2026-02-10 (2 Years) |
| Starting Equity | 100.0 USDT |
| **Final Equity** | **140.71 USDT** |
| Total Trades | 770 |

---

## Strategy Overview

### Timeframes

| Role | Timeframe |
|---|---|
| Execution + LTF Bands | 1 minute |
| Regime Detection + HTF Bands | 15 minutes (resampled from 1m) |

> **No-repaint alignment:** HTF features are shifted by 1 HTF bar (last fully closed candle), then forward-filled onto 1m bars.

---

### Indicators

**LTF Bands (1m)**

Long context uses `HIGH`-based bands with two centers:
- `bhe / bheU / bheL` — EMA on high ± stdev × mult
- `bhw / bhwU / bhwL` — WMA on high ± stdev × mult

Short context uses `LOW`-based bands with two centers:
- `ble / bleU / bleL` — EMA on low ± stdev × mult
- `blw / blwU / blwL` — WMA on low ± stdev × mult

**HTF Bands (15m on Close)**
- `bcs` = SMA(HTF close, length)
- `bcsU / bcsL` = bcs ± stdev(HTF close, length) × mult

---

### Regime Detection (HTF — last closed bar only)

Let `close_minus_sma = HTF_close − bcs`

| Regime | Condition |
|---|---|
| Long Breakout (`lbo`) | `rolling_MIN(close_minus_sma, x_lookback) > 0` |
| Short Breakout (`sbo`) | `rolling_MAX(close_minus_sma, x_lookback) < 0` |
| Mean Reversion (`mean`) | Not `lbo` and not `sbo` |

---

### Signals

**Breakout signals** (active when HTF is in breakout regime):
- `LBO_Long` — HTF `lbo` + LTF conditions + range breakout
- `SBO_Short` — HTF `sbo` + LTF conditions + range breakdown

**Mean-reversion signals** (active when HTF is in mean regime):
- `MEAN_Long` — mean regime + LTF conditions + interaction with HTF lower band
- `MEAN_Short` — mean regime + LTF conditions + interaction with HTF upper band

**Priority:** Breakout signals take precedence over mean-reversion signals.

---

### Execution Model

- **Entry:** Signal bar close (`process_orders_on_close = true`)
- **Position:** One position at a time
- **Exits:** OCO — Stop Loss + Take Profit
- **Intrabar fill logic:**
  - Long: SL if `low ≤ SL`, TP if `high ≥ TP`
  - Short: SL if `high ≥ SL`, TP if `low ≤ TP`
- **SL/TP same-bar collision:** controlled by `same_bar_fill_rule` (default: `"stop_first"`)

---

### Stops & Targets

- Breakout trades use `rr_breakout`
- Mean trades use `rr_mean`

| Direction | TP Formula |
|---|---|
| Long | `entry + RR × (entry − sl)` |
| Short | `entry − RR × (sl − entry)` |

---

### Position Sizing

- **Fixed qty (default):** `0.02 ETH` per trade (`use_fixed_qty = True`)
- Risk-based sizing is available but disabled by default
- PnL = `Δprice × qty × point_value` (`point_value = 1.0` for linear USDT-margined perp)

---

## Stop-Loss Statistics

| Metric | Value |
|---|---|
| Avg SL% | 0.3632% |
| Median SL% | 0.2465% |
| Min SL% | 0.0000352% |
| Max SL% | 5.5741% |

**Per signal type:**

| Signal | Count | Avg SL% | Min SL% | Max SL% |
|---|---|---|---|---|
| `LBO_Long` | 358 | 0.3503% | 0.0023% | 2.6548% |
| `SBO_Short` | 260 | 0.5514% | 0.0083% | 5.5741% |
| `MEAN_Long` | 74 | 0.0894% | 0.0000352% | 1.4905% |
| `MEAN_Short` | 78 | 0.0556% | 0.0004% | 0.2034% |

---

## Input Data Format

CSV with the following columns:

```
timestamp, open, high, low, close, volume
```

- `timestamp` must be parseable to `datetime` and set as the index
- Data must be clean and minute-aligned (check resample boundaries for timezone alignment)

---

## Configuration

| Parameter | Description |
|---|---|
| `length`, `mult` | Bollinger Band parameters |
| `htf_tf` | HTF timeframe string (e.g. `"15"`) |
| `x_lookback` | HTF regime detection lookback |
| `rr_breakout` | Risk/reward multiple for breakout trades |
| `rr_mean` | Risk/reward multiple for mean-reversion trades |
| `fixed_qty` | Constant ETH size per trade |
| `same_bar_fill_rule` | SL/TP collision handling (`"stop_first"` or `"tp_first"`) |

---

## Limitations

Results are **gross before costs** and do not model:

- Perpetual funding payments
- Spot–perp basis / premium / discount
- Mark price vs. last price differences
- Liquidation / margin mechanics
- Fees, spread, slippage, latency, partial fills, or order-book depth

Using spot candles as a perp proxy may overstate performance if funding, basis, or microstructure differences are material.
