# Bot py
My First bot
import MetaTrader5 as mt5
import pandas as pd
import time
import logging
from datetime import datetime

# CONFIGURATIONS
SYMBOL = "EURUSD"
FAST_MA = 10  # Fast moving average period
SLOW_MA = 30  # Slow moving average period
RSI_PERIOD = 14
RSI_THRESHOLD = 50
TIMEFRAME = mt5.TIMEFRAME_M1  # 1 minute
TRADE_RISK = 0.01  # 1% per trade
STOP_LOSS_PIPS = 20
TAKE_PROFIT_PIPS = 40
MAGIC_NUMBER = 270926  # Unique magic number for all bot trades

# LOGGING SETUP
logging.basicConfig(
    filename='trading_log.txt',
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s'
)

def log_trade(message):
    print(message)
    logging.info(message)

def initialize_mt5():
    if not mt5.initialize():
        print("MT5 initialize() failed")
        logging.error("MT5 initialize() failed")
        quit()
    print("MT5 initialized.")

def shutdown_mt5():
    mt5.shutdown()
    print("MT5 connection closed.")

def get_account_info():
    acc_info = mt5.account_info()
    if acc_info is None:
        raise Exception("Could not get account info")
    balance = acc_info.balance
    equity = acc_info.equity
    return balance, equity

def get_data(symbol, timeframe, n):
    """Fetch historical rates as Pandas DataFrame"""
    rates = mt5.copy_rates_from_pos(symbol, timeframe, 0, n)
    if rates is None or len(rates) < n:
        raise Exception("Not enough data to calculate indicators")
    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')
    return df

def calculate_indicators(df):
    """Adds fast and slow moving averages and RSI to DataFrame"""
    df['fast_ma'] = df['close'].rolling(window=FAST_MA).mean()
    df['slow_ma'] = df['close'].rolling(window=SLOW_MA).mean()
    df['rsi'] = ta_rsi(df['close'], RSI_PERIOD)
    return df

def ta_rsi(series, period):
    """Compute RSI as pandas Series"""
    delta = series.diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    return rsi

def lot_size(balance, price, stop_loss_pips):
    """Calculate position size to risk specified percent"""
    risk_amount = balance * TRADE_RISK
    ticks_per_pip = 10 if "JPY" in SYMBOL else 100000
    lot = risk_amount / (stop_loss_pips * (price / ticks_per_pip))
    lot = max(round(lot, 2), 0.01)  # Minimum 0.01 lot
    return lot

def get_positions():
    """Get all bot's open positions for EURUSD"""
    positions = mt5.positions_get(symbol=SYMBOL)
    # Filter by magic number to avoid duplicate bot trades
    return [p for p in positions if p.magic == MAGIC_NUMBER]

def check_crossover(df):
    """Check if the latest bar had an MA crossover"""
    if len(df) < SLOW_MA + 2:
        return None
    # Latest and previous values
    prev_fast = df['fast_ma'].iloc[-2]
    prev_slow = df['slow_ma'].iloc[-2]
    curr_fast = df['fast_ma'].iloc[-1]
    curr_slow = df['slow_ma'].iloc[-1]

    # Golden cross
    if prev_fast < prev_slow and curr_fast > curr_slow:
        return 'buy'
    # Death cross
    elif prev_fast > prev_slow and curr_fast < curr_slow:
        return 'sell'
    return None

def rsi_trend(df, direction):
    """Confirm trend by RSI.
       For buy: RSI > RSI_THRESHOLD, for sell: RSI < RSI_THRESHOLD
    """
    rsi = df['rsi'].iloc[-1]
    if direction == 'buy' and rsi > RSI_THRESHOLD:
        return True
    elif direction == 'sell' and rsi < RSI_THRESHOLD:
        return True
    return False

def open_trade(action, lot, price, sl_pips, tp_pips):
    """Send an order to open a trade"""
    deviation = 10
    # Calculate SL/TP prices
    point = mt5.symbol_info(SYMBOL).point
    if action == 'buy':
        sl = price - sl_pips * point * 10
        tp = price + tp_pips * point * 10
        order_type = mt5.ORDER_TYPE_BUY
    else:
        sl = price + sl_pips * point * 10
        tp = price - tp_pips * point * 10
        order_type = mt5.ORDER_TYPE_SELL

    request = {
        'action': mt5.TRADE_ACTION_DEAL,
        'symbol': SYMBOL,
        'volume': lot,
        'type': order_type,
        'price': price,
        'sl': sl,
        'tp': tp,
        'deviation': deviation,
        'magic': MAGIC_NUMBER,
        'comment': f"MA/RSI {action}",
        'type_time': mt5.ORDER_TIME_GTC,
        'type_filling': mt5.ORDER_FILLING_IOC
    }

    result = mt5.order_send(request)
    if result.retcode == mt5.TRADE_RETCODE_DONE:
        log_trade(f"{action.upper()} trade executed: lot={lot}, price={price}, SL={sl}, TP={tp}")
    else:
        log_trade(f"Trade failed [{action.upper()}]: retcode {result.retcode} - {result.comment}")

def main_loop():
    initialize_mt5()
    print("=" * 40)
    print(f"Running Forex Trading Bot for {SYMBOL}")
    print("=" * 40)

    try:
        while True:
            try:
                balance, equity = get_account_info()
                print(f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] Account balance: {balance:.2f}, equity: {equity:.2f}")

                positions = get_positions()
                print(f"Open positions: {len(positions)}")
                for pos in positions:
                    print(f"  {pos.type} {pos.volume} lots at {pos.price_open} - profit: {pos.profit}")

                # Get recent data for signal generation
                bars = SLOW_MA + 10
                df = get_data(SYMBOL, TIMEFRAME, bars)
                df = calculate_indicators(df)

                signal = check_crossover(df)
                if signal and rsi_trend(df, signal):
                    # Prevent duplicate trades
                    has_long = any(p.type == mt5.POSITION_TYPE_BUY for p in positions)
                    has_short = any(p.type == mt5.POSITION_TYPE_SELL for p in positions)
                    tick = mt5.symbol_info_tick(SYMBOL)
                    if signal == 'buy' and not has_long:
                        lot = lot_size(balance, tick.ask, STOP_LOSS_PIPS)
                        open_trade('buy', lot, tick.ask, STOP_LOSS_PIPS, TAKE_PROFIT_PIPS)
                    elif signal == 'sell' and not has_short:
                        lot = lot_size(balance, tick.bid, STOP_LOSS_PIPS)
                        open_trade('sell', lot, tick.bid, STOP_LOSS_PIPS, TAKE_PROFIT_PIPS)
                    else:
                        print("Duplicate or opposite trade prevented.")
                else:
                    print("No trade signal or RSI trend not confirmed.")

            except Exception as e:
                log_trade(f"Exception in trading loop: {e}")

            time.sleep(60)  # Run every minute

    except KeyboardInterrupt:
        print("Stopped by user.")
    finally:
        shutdown_mt5()

if __name__ == "__main__":
    main_loop()
