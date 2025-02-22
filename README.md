import requests
import time
import logging
import yfinance as yf
import pandas as pd
import numpy as np
import json
from datetime import datetime

# Konfigurasi logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# Token bot Telegram Anda
TOKEN = "7983528925:AAGHwH-g-Q73d4Gp2qbuWh49sbtx6Nq4uxY"
BASE_URL = f"https://api.telegram.org/bot{TOKEN}/"

# --- DAFTAR SAHAM REAL ---
REAL_STOCKS = [
    "BBCA.JK", "BBRI.JK", "BMRI.JK", "TLKM.JK", "ASII.JK",
    "UNVR.JK", "HMSP.JK", "ICBP.JK", "SMGR.JK", "CPIN.JK",
    "BBNI.JK", "BRPT.JK", "ACES.JK", "INDF.JK", "KLBF.JK",
    "PGAS.JK", "EXCL.JK", "ANTM.JK", "PTBA.JK", "TPIA.JK",
    "WIKA.JK", "INCO.JK", "MEDC.JK", "ADRO.JK", "AKPI.JK",
    "GGRM.JK", "TOWR.JK", "SCMA.JK", "PTPP.JK", "JSMR.JK",
    "SBTX.JK", "FASW.JK", "ELSA.JK", "ITMG.JK", "CTRA.JK",
    "MNCN.JK", "KRAS.JK"
]
ALL_STOCKS = REAL_STOCKS  # Hanya saham nyata

# Dictionary untuk menyimpan pilihan saham tiap user (satu pilihan per chat, disimpan sebagai string)
user_stock_selections = {}

# Variabel offset untuk getUpdates
offset = None

# --- FUNGSI TELEGRAM ---
def get_updates(offset):
    url = BASE_URL + "getUpdates"
    params = {"timeout": 50}
    if offset is not None:
        params["offset"] = offset
    try:
        response = requests.get(url, params=params)
        return response.json()
    except Exception as e:
        logging.error(f"Error getUpdates: {e}")
        return {}

def send_message(chat_id, text):
    url = BASE_URL + "sendMessage"
    data = {"chat_id": chat_id, "text": text}
    try:
        response = requests.post(url, data=data)
        if response.status_code != 200:
            logging.error(f"Error sending message: {response.text}")
    except Exception as e:
        logging.error(f"Error sending message: {e}")

def send_message_with_keyboard(chat_id, text, keyboard):
    url = BASE_URL + "sendMessage"
    data = {
        "chat_id": chat_id,
        "text": text,
        "reply_markup": json.dumps(keyboard)
    }
    try:
        response = requests.post(url, data=data)
        if response.status_code != 200:
            logging.error(f"Error sending message with keyboard: {response.text}")
    except Exception as e:
        logging.error(f"Error sending message with keyboard: {e}")

def send_document(chat_id, document_path, caption=""):
    url = BASE_URL + "sendDocument"
    files = {"document": open(document_path, "rb")}
    data = {"chat_id": chat_id, "caption": caption}
    try:
        response = requests.post(url, data=data, files=files)
        if response.status_code == 200:
            logging.info("Dokumen berhasil dikirim ke Telegram.")
        else:
            logging.error(f"Error sending document: {response.text}")
    except Exception as e:
        logging.error(f"Error sending document: {e}")

# --- INLINE KEYBOARD UNTUK PILIHAN SAHAM ---
def build_selection_keyboard(chat_id):
    """
    Membangun inline keyboard untuk saham nyata.
    Hanya satu saham yang dapat dipilih; awalnya tidak ada ceklis.
    Jika suatu saham telah dipilih, tombol tersebut akan menampilkan tanda centang (✅).
    Kelompokkan 2 tombol per baris, dan tambahkan tombol "Done".
    """
    selected_symbol = user_stock_selections.get(chat_id, None)
    buttons = []
    for symbol in ALL_STOCKS:
        if symbol == selected_symbol:
            text = "✅ " + symbol
        else:
            text = symbol
        buttons.append({"text": text, "callback_data": f"select:{symbol}"})
    # Kelompokkan tombol 2 per baris
    rows = []
    for i in range(0, len(buttons), 2):
        rows.append(buttons[i:i+2])
    # Tambahkan tombol "Done" di baris terpisah
    rows.append([{"text": "Done", "callback_data": "done"}])
    return {"inline_keyboard": rows}

def answer_callback(callback_id, text):
    url = BASE_URL + "answerCallbackQuery"
    data = {"callback_query_id": callback_id, "text": text}
    try:
        requests.post(url, data=data)
    except Exception as e:
        logging.error(f"Error answering callback: {e}")

def edit_reply_markup(chat_id, message_id, reply_markup):
    url = BASE_URL + "editMessageReplyMarkup"
    data = {"chat_id": chat_id, "message_id": message_id}
    if reply_markup is not None:
        data["reply_markup"] = json.dumps(reply_markup)
    else:
        data["reply_markup"] = ""
    try:
        requests.post(url, data=data)
    except Exception as e:
        logging.error(f"Error editing reply markup: {e}")

def process_callback(callback):
    data = callback.get("data", "")
    chat_id = callback["message"]["chat"]["id"]
    message_id = callback["message"]["message_id"]
    callback_id = callback["id"]

    if data.startswith("select:"):
        symbol = data.split(":", 1)[1]
        # Simpan pilihan tunggal sebagai string
        user_stock_selections[chat_id] = symbol
        keyboard = build_selection_keyboard(chat_id)
        edit_reply_markup(chat_id, message_id, keyboard)
        answer_callback(callback_id, f"Selected {symbol}")
    elif data == "done":
        selection = user_stock_selections.get(chat_id, None)
        if selection is None:
            answer_callback(callback_id, "No selection made.")
            send_message(chat_id, "No stock selected. Please select one stock.")
        else:
            edit_reply_markup(chat_id, message_id, None)
            send_message(chat_id, "Selection completed. Selected stock: " + selection)
            answer_callback(callback_id, "Selection done")
    else:
        answer_callback(callback_id, "Unknown action")

# --- ANALISIS INDIKATOR TECHNICAL & REKOMENDASI ---
def analyze_stock(symbol):
    """
    Mengambil data historis 3 bulan (interval harian) untuk analisis.
    Menghitung:
      - MA10
      - RSI (14 periode)
      - MACD dan Signal (EMA12, EMA26, EMA9)
    Mengembalikan nilai indikator beserta harga dan persentase perubahan.
    """
    try:
        ticker = yf.Ticker(symbol)
        data = ticker.history(period="3mo", interval="1d")
        if data.empty or len(data) < 26:
            return None

        current_price = data['Close'].iloc[-1]
        if len(data) >= 2:
            prev_price = data['Close'].iloc[-2]
            change = current_price - prev_price
            percent_change = (change / prev_price * 100) if prev_price != 0 else 0
        else:
            change = 0
            percent_change = 0

        ma10 = data['Close'].rolling(window=10).mean().iloc[-1]

        delta = data['Close'].diff()
        gain = delta.clip(lower=0)
        loss = -delta.clip(upper=0)
        avg_gain = gain.rolling(window=14, min_periods=14).mean()
        avg_loss = loss.rolling(window=14, min_periods=14).mean()
        rs = avg_gain / avg_loss
        rsi = 100 - (100 / (1 + rs))
        rsi_current = rsi.iloc[-1]

        ema12 = data['Close'].ewm(span=12, adjust=False).mean()
        ema26 = data['Close'].ewm(span=26, adjust=False).mean()
        macd = ema12 - ema26
        signal = macd.ewm(span=9, adjust=False).mean()
        macd_current = macd.iloc[-1]
        signal_current = signal.iloc[-1]

        return {
            "symbol": symbol,
            "current_price": current_price,
            "change": change,
            "percent_change": percent_change,
            "ma10": ma10,
            "rsi": rsi_current,
            "macd": macd_current,
            "signal": signal_current
        }
    except Exception as e:
        logging.error(f"Error analyzing {symbol}: {e}")
        return None

def compute_recommendation(result):
    """
    Menghitung buy probability berdasarkan:
      - RSI: diasumsikan RSI 30 mendapat skor 1 (oversold) dan RSI 70 mendapat skor 0.
      - MACD: jika MACD > Signal, skor 1.
      - MA: jika harga > MA10, skor 1.
    Kemudian rata-rata skor (dalam persen) dihitung dan
    ditentukan rekomendasi beserta harga beli dan jual.
    """
    rsi = result["rsi"]
    rsi_score = np.clip((70 - rsi) / 40, 0, 1)
    macd_score = 1 if result["macd"] > result["signal"] else 0
    ma_score = 1 if result["current_price"] > result["ma10"] else 0
    prob = (rsi_score + macd_score + ma_score) / 3 * 100

    if prob >= 60:
        rec = "Strong Buy"
        buy_price = result["current_price"] * 0.98
        sell_price = result["current_price"] * 1.05
    elif prob >= 50:
        rec = "Buy"
        buy_price = result["current_price"] * 0.99
        sell_price = result["current_price"] * 1.03
    elif prob >= 40:
        rec = "Hold"
        buy_price = None
        sell_price = None
    else:
        rec = "Sell"
        buy_price = None
        sell_price = result["current_price"] * 0.95

    return {"buy_prob": prob, "recommendation": rec, "buy_price": buy_price, "sell_price": sell_price}

# --- PROSES PESAN MASUK ---
def process_message(message):
    chat = message.get("chat", {})
    chat_id = chat.get("id")
    if not chat_id:
        return

    text = message.get("text", "")
    if text.startswith("/start"):
        welcome = (
            "Selamat datang di Bot Analisa Saham Indonesia!\n\n"
            "Perintah yang tersedia:\n"
            "/select - Pilih saham dengan tombol (hanya satu pilihan)\n"
            "/analyze - Lihat analisis teknikal lengkap beserta rekomendasi (harga beli/jual)\n\n"
            "Jika belum memilih, default akan digunakan (semua saham nyata)."
        )
        send_message(chat_id, welcome)
        if chat_id not in user_stock_selections:
            user_stock_selections[chat_id] = None
    elif text.startswith("/select"):
        keyboard = build_selection_keyboard(chat_id)
        prompt = "Klik tombol di bawah untuk memilih saham (hanya satu pilihan):"
        send_message_with_keyboard(chat_id, prompt, keyboard)
    elif text.startswith("/analyze"):
        selection = user_stock_selections.get(chat_id, None)
        response_lines = ["Hasil Analisa Saham:"]
        now = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        response_lines.append(f"Waktu: {now}\n")
        if selection is not None:
            result = analyze_stock(selection)
            if result is None:
                response_lines.append(f"{selection}: Data tidak tersedia.")
            else:
                rec_data = compute_recommendation(result)
                line = (
                    f"📌 {result['symbol']}\n"
                    f"💰 Price: {result['current_price']:.2f}\n"
                    f"{'📈' if result['change'] >= 0 else '📉'} Change: {result['change']:.2f} ({result['percent_change']:.2f}%)\n"
                    f"📊 MA10: {result['ma10']:.2f}\n"
                    f"🔖 RSI: {result['rsi']:.2f}\n"
                    f"⚡ MACD: {result['macd']:.2f} (Signal: {result['signal']:.2f})\n"
                    f"✅ Buy Prob: {rec_data['buy_prob']:.1f}%\n"
                    f"📝 Recommendation: {rec_data['recommendation']}"
                )
                if rec_data['buy_price'] is not None:
                    line += f"\n🛒 Buy @ {rec_data['buy_price']:.2f}"
                if rec_data['sell_price'] is not None:
                    line += f"\n💹 Sell @ {rec_data['sell_price']:.2f}"
                response_lines.append(line)
        else:
            # Jika belum ada pilihan, tampilkan analisis untuk semua saham nyata
            for symbol in ALL_STOCKS:
                result = analyze_stock(symbol)
                if result is None:
                    response_lines.append(f"{symbol}: Data tidak tersedia.")
                else:
                    rec_data = compute_recommendation(result)
                    line = (
                        f"📌 {result['symbol']}\n"
                        f"💰 Price: {result['current_price']:.2f}\n"
                        f"{'📈' if result['change'] >= 0 else '📉'} Change: {result['change']:.2f} ({result['percent_change']:.2f}%)\n"
                        f"📊 MA10: {result['ma10']:.2f}\n"
                        f"🔖 RSI: {result['rsi']:.2f}\n"
                        f"⚡ MACD: {result['macd']:.2f} (Signal: {result['signal']:.2f})\n"
                        f"✅ Buy Prob: {rec_data['buy_prob']:.1f}%\n"
                        f"📝 Recommendation: {rec_data['recommendation']}"
                    )
                    if rec_data['buy_price'] is not None:
                        line += f"\n🛒 Buy @ {rec_data['buy_price']:.2f}"
                    if rec_data['sell_price'] is not None:
                        line += f"\n💹 Sell @ {rec_data['sell_price']:.2f}"
                    response_lines.append(line)
        send_message(chat_id, "\n\n".join(response_lines))
    else:
        send_message(chat_id, "Perintah tidak dikenal. Gunakan /start, /select, atau /analyze.")

# --- LOOP UTAMA ---
def main():
    global offset
    logging.info("Bot sedang berjalan...")
    while True:
        updates = get_updates(offset)
        if updates.get("result"):
            for item in updates["result"]:
                offset = item["update_id"] + 1
                if "message" in item:
                    process_message(item["message"])
                elif "callback_query" in item:
                    process_callback(item["callback_query"])
        time.sleep(1)

if __name__ == "__main__":
    main()
