# Mikanoimport os
import telebot
import pandas as pd
import numpy as np
import requests
from dotenv import load_dotenv

# تحميل المتغيرات السرية من ملف .env
load_dotenv()
BOT_TOKEN = os.getenv("BOT_TOKEN")

if not BOT_TOKEN:
    raise ValueError("خطأ: لم يتم العثور على BOT_TOKEN في ملف .env!")

bot = telebot.TeleBot(BOT_TOKEN)

# دالة لجلب بيانات السوق من Binance كمثال
def get_market_data(symbol="BTCUSDT", interval="1h", limit=100):
    url = f"https://binance.com{symbol.upper()}&interval={interval}&limit={limit}"
    try:
        response = requests.get(url).json()
        df = pd.DataFrame(response, columns=['time', 'open', 'high', 'low', 'close', 'volume', 'close_time', 'qav', 'num_trades', 'taker_base', 'taker_quote', 'ignore'])
        df = df.astype(float)
        return df
    except Exception as e:
        return None

# دالة حساب المؤشرات الفنية (الزخم، الاتجاه، السيولة، والشموع)
def analyze_market(df):
    if df is None or df.empty:
        return "⚠️ عذراً، تعذر جلب البيانات للرمز المدخل."
    
    last_row = df.iloc[-1]
    prev_row = df.iloc[-2]
    
    # 1. الاتجاه (Moving Average 20)
    ma20 = df['close'].rolling(window=20).mean().iloc[-1]
    current_price = last_row['close']
    trend = "📈 صاعد (فوق MA20)" if current_price > ma20 else "📉 هابط (تحت MA20)"
    
    # 2. الزخم (RSI 14 تقريبي)
    delta = df['close'].diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=14).mean().iloc[-1]
    loss = (-delta.where(delta < 0, 0)).rolling(window=14).mean().iloc[-1]
    rs = gain / (loss if loss != 0 else 1)
    rsi = round(100 - (100 / (1 + rs)), 2)
    
    if rsi > 70: momentum = f"🔥 شراء مفرط ({rsi}) - قد يعكس لأسفل"
    elif rsi < 30: momentum = f"🧊 بيع مفرط ({rsi}) - قد يعكس لأعلى"
    else: momentum = f"⚖️ معتدل ({rsi})"
    
    # 3. السيولة وعمليات البيع والشراء
    avg_volume = df['volume'].mean()
    current_volume = last_row['volume']
    liquidity = "💧 سيولة عالية (فوق المعدل)" if current_volume > avg_volume else "🏜️ سيولة منخفضة"
    
    # تقدير قوى البيع والشراء بناءً على الشمعة الحالية
    if current_price > last_row['open']:
        market_force = "🟢 قوى الشراء تسيطر على الجلسة الحالية"
    else:
        market_force = "🔴 قوى البيع تسيطر على الجلسة الحالية"
        
    # 4. تحليل الشموع اليابانية (نماذج بسيطة)
    body = abs(current_price - last_row['open'])
    total_range = last_row['high'] - last_row['low']
    upper_wick = last_row['high'] - max(current_price, last_row['open'])
    lower_wick = min(current_price, last_row['open']) - last_row['low']
    
    candle_type = "شمعة عادية"
    if total_range > 0:
        if lower_wick > (body * 2) and upper_wick < (body * 0.5):
            candle_type = "🔨 مطرقة (Hammer) - إشارة انعكاس صاعد محتملة"
        elif upper_wick > (body * 2) and lower_wick < (body * 0.5):
            candle_type = "☄️ شهاب (Shooting Star) - إشارة انعكاس هابط محتملة"
        elif body / total_range < 0.1:
            candle_type = "⏳ دوجي (Doji) - حيرة في السوق وعدم استقرار"

    # تجميع التقرير للتنبيه
    report = (
        f"📊 **تقرير التنبيه الفني لزوج {df.filename if hasattr(df,'filename') else ''}**\n\n"
        f"💵 **السعر الحالي:** ${current_price:,.2f}\n"
        f"🧭 **الاتجاه العام:** {trend}\n"
        f"⚡ **مؤشر الزخم (RSI):** {momentum}\n"
        f"🌊 **السيولة الحالية:** {liquidity}\n"
        f"⚔️ **حركة السوق:** {market_force}\n"
        f"🕯️ **تحليل الشمعة الأخيرة:** {candle_type}\n\n"
        f"💡 *مقترح البوت:* "
    )
    if "صاعد" in trend and rsi < 65:
        report += "الظروف تدعم استمرار الصعود (فرصة شراء محتملة بحذر)."
    elif "هابط" in trend and rsi > 35:
        report += "الظروف تدعم استمرار الهبوط (تجنب الدخول حالياً)."
    else:
        report += "السوق في منطقة حيرة، يفضل الانتظار لتأكيد الاتجاه."
        
    return report

# أوامر البوت في تيليجرام
@bot.message_handler(commands=['start', 'help'])
def send_welcome(message):
    bot.reply_to(message, "مرحباً بك في بوت التنبيهات الفنية المتقدم! 🤖📊\n\nأرسل رمز العملة لتحليله فوراً (مثال: `btc` أو `eth` أو `sol`).")

@bot.message_handler(func=lambda message: True)
def handle_analysis(message):
    symbol = message.text.strip().upper()
    # تحويل الاختصارات لزوج الدولار الرقمي في بينانس
    if not symbol.endswith("USDT"):
        search_symbol = f"{symbol}USDT"
    else:
        search_symbol = symbol
        
    bot.reply_to(message, f"🔄 جاري سحب البيانات وتحليل {search_symbol}...")
    
    df = get_market_data(search_symbol)
    report = analyze_market(df)
    
    bot.reply_to(message, report, parse_mode="Markdown")

# تشغيل البوت باستمرار
print("🤖 البوت يعمل الآن بنجاح...")
bot.infinity_polling()
