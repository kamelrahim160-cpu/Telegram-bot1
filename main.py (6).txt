
import os
import json
import asyncio
import logging
from datetime import datetime, date
from telegram import Update
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes
from google import genai

# CONFIG
BOT_TOKEN = os.getenv("BOT_TOKEN") or os.getenv("TELEGRAM_BOT_TOKEN")
GEMINI_API_KEY = os.getenv("GEMINI_API_KEY")
VODAFONE_CASH = "01094659074"
FREE_LIMIT = 5
USAGE_FILE = "usage.json"

logging.basicConfig(level=logging.INFO)

# Gemini client - new SDK
client = genai.Client(api_key=GEMINI_API_KEY)

def load_usage():
    if not os.path.exists(USAGE_FILE):
        return {}
    try:
        with open(USAGE_FILE, 'r', encoding='utf-8') as f:
            return json.load(f)
    except:
        return {}

def save_usage(data):
    # FIXED: correct order file, data
    with open(USAGE_FILE, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def can_ask(user_id):
    data = load_usage()
    today = str(date.today())
    uid = str(user_id)
    if uid not in data:
        return True, 0
    if data[uid].get('date') != today:
        return True, 0
    used = data[uid].get('count', 0)
    paid = data[uid].get('paid', False)
    if paid:
        return True, used
    return used < FREE_LIMIT, used

def record_usage(user_id):
    data = load_usage()
    today = str(date.today())
    uid = str(user_id)
    if uid not in data or data[uid].get('date') != today:
        data[uid] = {'date': today, 'count': 1, 'paid': False}
    else:
        data[uid]['count'] = data[uid].get('count', 0) + 1
    save_usage(data)
    return data[uid]['count']

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    msg = f"""أهلا بيك في بوت المذاكرة الذكي 📚

ليك {FREE_LIMIT} أسئلة مجانية يوميا.
بعد كده الاشتراك بـ 50ج فودافون كاش على الرقم:
{VODAFONE_CASH}

ابعت سؤالك في أي مادة وهجاوبك فورا."""
    await update.message.reply_text(msg)

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    question = update.message.text
    
    allowed, used = can_ask(user_id)
    if not allowed:
        await update.message.reply_text(
            f"خلصت الـ {FREE_LIMIT} أسئلة المجانية بتاعة النهاردة.\n"
            f"عشان تكمل، حول 50ج فودافون كاش على {VODAFONE_CASH}\n"
            f"وبعتلي سكرين التحويل وهفعللك الاشتراك فورا ✅"
        )
        return

    try:
        response = client.models.generate_content(
            model="gemini-2.0-flash",
            contents=f"أنت مساعد مذاكرة شاطر، جاوب بوضوح وباختصار: {question}"
        )
        answer = response.text
        count = record_usage(user_id)
        remaining = FREE_LIMIT - count if not load_usage().get(str(user_id), {}).get('paid') else "مفتوح"
        await update.message.reply_text(f"{answer}\n\n---\nمتبقي لك: {remaining} أسئلة مجانية اليوم")
    except Exception as e:
        logging.error(e)
        await update.message.reply_text("حصل مشكلة بسيطة، جرب تاني بعد دقيقة")

def main():
    if not BOT_TOKEN or not GEMINI_API_KEY:
        print("BOT_TOKEN و GEMINI_API_KEY لازم يكونوا موجودين")
        return
    # prevent duplicate polling conflict message
    print("Bot with monetization is running...")
    app = Application.builder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    app.run_polling()

if __name__ == "__main__":
    main()
