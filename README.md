# Bot py
My First botl
import requests
import pandas as pd
import time
from datetime import datetime
import logging
import math

# =========== CONFIG ===========

# [1] API config (GET YOUR FREE API KEY!)
API_KEY = "<YOUR_API_KEY_HERE>"  # e.g., https://financialmodelingprep.com/developer/docs/
SYMBOL = "EURUSD"  # Used by most APIs as 'EURUSD', double-check for your API
API_URL = f"https://financialmodelingprep.com/api/v3/historical-chart/1min/{SYMBOL}?apikey={API_KEY}"

# [2] Strategy parameters
FAST_MA = 10
SLOW_MA = 30
RSI_PERIOD = 14
RSI_THRESHOLD = 50
STOP_LOSS_PIPS = 20
TAKE_PROFIT_PIPS = 40
RISK_PER_TRADE = 0.01

# [3] Simulation
INITIAL_BALANCE = 10000.0  # USD
MIN_LOT = 0.01

# [4] Log file
logging.basicConfig(filename="sim_trading_log.txt", level=logging.INFO, 
                    format='%(asctime)s %(levelname)s %(message)s')

# =========== STATE ===========

class Position:
    def __init__(self, direction, open_price, lot, sl, tp, open_time):
        self.direction = direction  # "buy" or "sell"
        self.open_price = open_price
        self.lot = lot
        self.sl = sl
        self.tp = tp
        self.open_time = open_time
        self.close_price = None
        self.close_time = None
        self.profit = 0

# Single open position (None if no trade open)
open_position = None
balance = INITIAL_BALANCE
trade_history = []

# =========== INDICATORS ===========

def calc_sma(series, period):
    return series.rolling(window=period).mean()

def calc_rsi(series, period):
    delta = series.diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    return rsi

# =========== UTILS ===========

def get_bars(n=50):
    """Fetches the last n 1-minute bars for EURUSD."""
    try:
        resp = requests.get(API_URL)
        data = resp.json()
        if not data or isinstance(data, dict) and data.get('Error Message'):
            raise Exception("API error or quota exceeded")
        df = pd.DataFrame(data)
        df = df.sort_values('date')
        df = df.tail(n)  # last n bars in time order
        df['close'] = df['close'].astype(float)
        return df.reset_index(drop=True)
    except Exception as e:
        logging.error(f"Failed to download/fetch bar data: {e}")
        return None

def log_and_print(msg):
    print(msg)
    logging.info(msg)

def get_lot_size(balance, price, stoploss_pips):
    # Example for EURUSD: 1 lot = 100,000 units, pip = 0.0001
    risk_dollars = balance * RISK_PER_TRADE
    pip_value_per_lot = 10  # For EURUSD (1 lot is $10/pip)
    lot = max(risk_dollars / (stoploss_pips * pip_value_per_lot), MIN_LOT)
    return round(lot, 2)

def simulate_trade_close(position, current_price, current_time):
    global balance
    if position.direction == "buy":
        # SL hit
        if current_price <= position.sl:
            position.close_price = position.sl
            note = "Stop Loss hit"
        # TP hit
        elif current_price >= position.tp:
            position.close_price = position.tp
            note = "Take Profit hit"
        else:
            return False  # Still open
    else:  # sell
        if current_price >= position.sl:
            position.close_price = position.sl
            note = "Stop Loss hit"
        elif current_price <= position.tp:
            position.close_price = position.tp
            note = "Take Profit hit"
        else:
            return False  

    position.close_time = current_time
    position.profit = calc_profit(position)
    trade_history.append(position)
    balance += position.profit
    log_and_print(f"Closed {position.direction.upper()} at {position.close_price:.5f} ({note}), Profit=${position.profit:.2f}")
    logging.info(f"Account balance after close: {balance:.2f}")
    return True

def calc_profit(position):
    # Pip value per lot: for EURUSD, 1 lot = $10 per pip, 1 pip = 0.0001
    pip_value = 10  # Per lot
    pips = 0
    if position.direction == "buy":
        pips = (position.close_price - position.open_price) / 0.0001
    else:
        pips = (position.open_price - position.close_price) / 0.0001
    return position.lot * pip_value * pips

def print_account_status():
    print(f"Balance: ${balance:.2f} | Open position: {'Yes' if open_position else 'No'}")
    if open_position:
        print(f"  {open_position.direction.upper()} | Open@ {open_position.open_price:.5f} | SL: {open_position.sl:.5f} | TP: {open_position.tp:.5f} | Lot: {open_position.lot}")

# =========== MAIN STRATEGY ===========

def main_loop():
    global open_position, balance
    print("Forex Simulated Trading Bot (EURUSD, 1min bars)")
    log_and_print(f"Starting bal: {balance:.2f}")
    while True:
        try:
            df = get_bars(SLOW_MA+RSI_PERIOD+5)
            if df is None or len(df) < SLOW_MA+2:
                print("Waiting for sufficient data...")
                time.sleep(60)
                continue

            # --- Compute signals
            df["fast_ma"] = calc_sma(df["close"], FAST_MA)
            df["slow_ma"] = calc_sma(df["close"], SLOW_MA)
            df["rsi"] = calc_rsi(df["close"], RSI_PERIOD)

            # Prepare for signal extraction
            prev_fast = df["fast_ma"].iloc[-2]
            prev_slow = df["slow_ma"].iloc[-2]
            now_fast = df["fast_ma"].iloc[-1]
            now_slow = df["slow_ma"].iloc[-1]

            signal = None
            # Golden cross (buy, fast crosses up slow)
            if prev_fast < prev_slow and now_fast > now_slow:
                signal = "buy"
            # Death cross (sell, fast crosses down slow)
            elif prev_fast > prev_slow and now_fast < now_slow:
                signal = "sell"

            # RSI filter
            now_rsi = df["rsi"].iloc[-1]
            rsi_ok = False
            if signal == "buy" and now_rsi > RSI_THRESHOLD:
                rsi_ok = True
            elif signal == "sell" and now_rsi < RSI_THRESHOLD:
                rsi_ok = True

            # Current bar info
            price = float(df["close"].iloc[-1])
            date_time = df["date"].iloc[-1]

            # ========== TRADE EXECUTION/SIMULATION ==========
            print_account_status()
            if open_position:
                # Check if SL/TP hit
                closed = simulate_trade_close(open_position, price, date_time)
                if closed:
                    open_position = None

            if not open_position and signal and rsi_ok:
                lot = get_lot_size(balance, price, STOP_LOSS_PIPS)
                if signal == "buy":
                    sl = price - STOP_LOSS_PIPS * 0.0001
                    tp = price + TAKE_PROFIT_PIPS * 0.0001
                else:
                    sl = price + STOP_LOSS_PIPS * 0.0001
                    tp = price - TAKE_PROFIT_PIPS * 0.0001
                open_position = Position(signal, price, lot, sl, tp, date_time)
                log_and_print(f"Opened {signal.upper()} @ {price:.5f} | Lot {lot} | SL {sl:.5f} | TP {tp:.5f}")
            elif not signal or not rsi_ok:
                print("No new signal or RSI not in agreement.")
            
        except Exception as e:
            logging.error(f"Error in main loop: {e}")
            print(f"Error: {e}")
        time.sleep(60)

if __name__ == "__main__":
    main_loop()
