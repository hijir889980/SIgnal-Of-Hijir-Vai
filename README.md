import time
import yfinance as yf
import pandas as pd
import ta
from telegram import Bot

TOKEN = "7123656829:AAG1LXgMUugPBfaHgGTjC_lnMsHKhABXmsY"
CHAT_ID = 7702783416
INTERVAL = 60  # every 1 minute

PAIRS = [
    "EURUSD=X", "GBPUSD=X", "USDJPY=X", "USDCHF=X", "AUDUSD=X",
    "NZDUSD=X", "USDCAD=X", "EURGBP=X", "EURJPY=X", "GBPJPY=X",
    "EURCHF=X", "AUDJPY=X", "NZDJPY=X", "GBPCHF=X", "EURCAD=X",
    "CADJPY=X", "CHFJPY=X", "AUDCHF=X", "NZDCHF=X", "GBPAUD=X"
]

bot = Bot(token=TOKEN)

def fetch_data(symbol):
    try:
        df = yf.download(tickers=symbol, period="1d", interval="1m")
        df.dropna(inplace=True)
        return df
    except:
        return None

def calculate_indicators(df):
    df['rsi'] = ta.momentum.RSIIndicator(df['Close'], window=14).rsi()
    macd = ta.trend.MACD(df['Close'])
    df['macd'] = macd.macd()
    df['signal'] = macd.macd_signal()
    return df

def generate_signal(df):
    latest = df.iloc[-1]
    if latest['rsi'] < 30 and latest['macd'] > latest['signal']:
        return "CALL"
    elif latest['rsi'] > 70 and latest['macd'] < latest['signal']:
        return "PUT"
    else:
        return None

def send_signal(pair, signal):
    symbol = pair.replace("=X", "")
    message = f"""
🔥 Quotex OTC Signal 🔥
Pair: {symbol}
Signal: {signal}
Indicators: RSI + MACD
Time: Now
    """
    bot.send_message(chat_id=CHAT_ID, text=message)
    print(f"Sent: {symbol} => {signal}")

def main():
    while True:
        for pair in PAIRS:
            df = fetch_data(pair)
            if df is not None and len(df) > 30:
                df = calculate_indicators(df)
                signal = generate_signal(df)
                if signal:
                    send_signal(pair, signal)
        time.sleep(INTERVAL)

if __name__ == "__main__":
    main()
python-telegram-bot==13.15
yfinance
ta
pandas
services:
  - type: worker
    name: signal-of-hijir-bot
    env: python
    plan: free
    buildCommand: "pip install -r requirements.txt"
    startCommand: "python main.py"
    autoDeploy: true
