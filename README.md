# multitf-bollinger-switching
Multi-timeframe Bollinger Band switching strategy for crypto backtesting (Python &amp; Pine Script)

CUSTOM BB SWITCHING STRATEGY (LTF 1m + HTF 15m) — ETHUSDT PERP (spot data proxy)
Backtesting Period: 2024-02-10 to 2026-02-10 (2 Years)

OVERVIEW  
This repo backtests a Bollinger Band “regime switching” strategy that trades ETHUSDT perpetual using 1-minute execution logic, while using a 15-minute higher timeframe to decide whether the market is in a breakout regime or a mean-reversion regime.

INSTRUMENT & DATA ASSUMPTION (IMPORTANT)  
• Trading instrument (assumed): ETHUSDT perpetual (linear USDT-margined)  
• Data used: ETHUSDT spot 1-minute OHLCV  
• Core assumption: spot and perp move identically (basis ≈ 0), so spot candles are a proxy for perp execution

NOT MODELED (so results are idealized)  
• Perp funding payments  
• Spot–perp basis / premium / discount  
• Mark price vs last price differences  
• Liquidation / margin mechanics  
• Fees, spread, slippage, latency, partial fills, order-book depth

TIMEFRAMES  
• LTF (execution + LTF bands): 1 minute  
• HTF (regime + HTF BB): 15 minutes, resampled from 1 minute  
• No-repaint alignment: HTF features are shifted by 1 HTF bar (last fully closed candle), then forward-filled to 1m bars

INDICATORS  
1) LTF Bands (1m)  
• Long context uses HIGH-based bands with two centers: EMA and WMA  
  – bhe/bheU/bheL (EMA on high ± stdev*mult)  
  – bhw/bhwU/bhwL (WMA on high ± stdev*mult)  
• Short context uses LOW-based bands with two centers: EMA and WMA  
  – ble/bleU/bleL (EMA on low ± stdev*mult)  
  – blw/blwU/blwL (WMA on low ± stdev*mult)

2) HTF Bands (15m) on CLOSE  
• bcs = SMA(HTF close, length)  
• bcsU/L = bcs ± stdev(HTF close, length) * mult

REGIME DETECTION (HTF, last closed bar only)  
Let close_minus_sma = HTF_close − bcs  
• Long breakout regime (lbo): rolling MIN(close_minus_sma, x_lookback) > 0  
• Short breakout regime (sbo): rolling MAX(close_minus_sma, x_lookback) < 0  
• Otherwise mean regime: mean = not lbo and not sbo

SIGNALS (4 TYPES, evaluated on 1m)  
Breakout signals (only when HTF indicates breakout):  
• LBO_Long: HTF lbo + LTF conditions + range breakout  
• SBO_Short: HTF sbo + LTF conditions + range breakdown

Mean-reversion signals (only when HTF indicates mean):  
• MEAN_Long: mean regime + LTF mean conditions + interaction with HTF lower band area  
• MEAN_Short: mean regime + LTF mean conditions + interaction with HTF upper band area  
Note: In the code, bcs/bcsU/bcsL already represent the last closed HTF bar because HTF series are shifted before alignment.

SIGNAL PRIORITY  
• Breakout first (LBO_Long / SBO_Short), then Mean (MEAN_Long / MEAN_Short)

EXECUTION MODEL (BACKTEST)  
• Entry: at signal bar CLOSE (TradingView-like process_orders_on_close=true)  
• One position max at a time  
• OCO exits: Stop Loss + Take Profit  
• Intrabar fill logic using OHLC:  
  – Long: SL if low <= SL, TP if high >= TP  
  – Short: SL if high >= SL, TP if low <= TP  
• If SL and TP both hit in the same bar: controlled by same_bar_fill_rule (default “stop_first”)

STOPS & TARGETS  
• Breakout trades use rr_breakout  
• Mean trades use rr_mean  
TP formulas:  
• Long: tp = entry + RR * (entry − sl)  
• Short: tp = entry − RR * (sl − entry)

POSITION SIZING  
• Fixed quantity (no risk sizing): fixed_qty = 0.02 ETH per trade  
• Risk-based sizing exists but is disabled when use_fixed_qty=True  
• PnL is computed as (Δprice) * qty * point_value (point_value=1.0 for linear USDT-margined perp proxy)

INPUT DATA FORMAT  
CSV columns: timestamp, open, high, low, close, volume  
timestamp must be parseable to datetime and set as index.

RESULT (GROSS, BEFORE COSTS)  
Run output:  
• Final equity: 140.7137146302667 (starting equity = 100.0)  
• Number of trades: 770  

SL% statistics (distance from entry to SL, as % of entry):  
• Avg SL%: 0.3632388693%  
• Median SL%: 0.2464723612%  
• Min SL%: 0.0000351602%  
• Max SL%: 5.5741477136%

Per-tag SL% (avg, min, max):  
• LBO_Long (n=358): avg 0.3502627586%, min 0.0022935254%, max 2.6548372801%  
• SBO_Short (n=260): avg 0.5513526056%, min 0.0082718973%, max 5.5741477136%  
• MEAN_Long (n=74): avg 0.0893544247%, min 0.0000351602%, max 1.4904738133%  
• MEAN_Short (n=78): avg 0.0555891912%, min 0.0003742867%, max 0.2033872170%

IMPORTANT NOTES  
• Results exclude fees, spread, slippage, funding, and basis effects.  
• Using spot candles as a perp proxy can overstate performance if funding/basis or microstructure matters.  
• Resample boundaries depend on timestamp alignment/timezone; ensure your 1m data is clean and minute-aligned.

CONFIG QUICK MAP  
• length, mult: BB parameters  
• htf_tf (“15”), x_lookback: HTF regime detection  
• rr_breakout, rr_mean: target multiples  
• fixed_qty: constant ETH size per trade  
• same_bar_fill_rule: SL/TP collision handling

RESEARCH ONLY — NOT FINANCIAL ADVICE
