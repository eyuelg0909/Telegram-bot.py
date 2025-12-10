import logging
import random
import sqlite3
from datetime import datetime
from telegram import (
    Update,
    InlineKeyboardButton,
    InlineKeyboardMarkup,
    ReplyKeyboardRemove
)
from telegram.ext import (
    ApplicationBuilder,
    CommandHandler,
    MessageHandler,
    CallbackQueryHandler,
    ConversationHandler,
    ContextTypes,
    filters
)
from apscheduler.schedulers.asyncio import AsyncIOScheduler
import os
import asyncio

# ----------------------------
# CONFIG (Render environment variables)
# ----------------------------
BOT_TOKEN = os.getenv("BOT_TOKEN", "8526085694:AAH-kRVcN8Goxv3-axNu6QMzNP6-ykTevi0")
ADMIN_CHAT_ID = int(os.getenv("ADMIN_CHAT_ID", "454246152"))
PUBLIC_CHAT_ID = int(os.getenv("PUBLIC_CHAT_ID", "-1003311097361"))
DB_NAME = "lottery.db"

# ----------------------------
# DATABASE INITIALIZATION
# ----------------------------
conn = sqlite3.connect(DB_NAME, check_same_thread=False)
cursor = conn.cursor()

def init_db():
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            username TEXT,
            name TEXT,
            phone TEXT,
            category TEXT,
            item TEXT,
            price INTEGER,
            payment_method TEXT,
            receipt TEXT UNIQUE,
            lotto TEXT UNIQUE,
            winner INTEGER DEFAULT 0
        )
    """)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS valid_receipts (
            receipt TEXT PRIMARY KEY
        )
    """)
    conn.commit()
    print("[OK] Database ready.")

def add_valid_receipt(receipt: str):
    try:
        cursor.execute("INSERT INTO valid_receipts (receipt) VALUES (?)", (receipt,))
        conn.commit()
    except sqlite3.IntegrityError:
        pass

def receipt_exists(receipt: str):
    cursor.execute("SELECT 1 FROM users WHERE receipt=?", (receipt,))
    return cursor.fetchone() is not None

def generate_unique_lotto(category):
    while True:
        lotto_number = f"LC-{random.randint(100000, 999999)}"
        cursor.execute("SELECT 1 FROM users WHERE lotto=? AND category=?", (lotto_number, category))
        if cursor.fetchone() is None:
            return lotto_number

# ----------------------------
# STATES
# ----------------------------
NAME, PHONE, CATEGORY, PAYMENT_METHOD, RECEIPT, CONFIRM = range(6)

# ----------------------------
# REGISTRATION FLOW
# ----------------------------
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.type != "private":
        await update.message.reply_text("⚠️ Please DM me in private to join the lottery.")
        return ConversationHandler.END
    await update.message.reply_text(
        "👋 Welcome to *LuckyCarLottery!*\n\nPlease enter your full name:",
        parse_mode="Markdown",
        reply_markup=ReplyKeyboardRemove()
    )
    return NAME

async def get_name(update: Update, context: ContextTypes.DEFAULT_TYPE):
    context.user_data["name"] = update.message.text.strip()
    await update.message.reply_text("📱 Now enter your phone number (+251XXXXXXXXX or 09XXXXXXXXX):")
    return PHONE

async def get_phone(update: Update, context: ContextTypes.DEFAULT_TYPE):
    phone = update.message.text.strip()
    if not (phone.startswith("+251") or phone.startswith("09")) or not phone.replace("+", "").isdigit():
        await update.message.reply_text("⚠️ Invalid phone. Use +251XXXXXXXXX or 09XXXXXXXXX.")
        return PHONE
    context.user_data["phone"] = phone

    keyboard = [
        [InlineKeyboardButton("🚗 Car Lottery (1000 Birr)", callback_data="Car")],
        [InlineKeyboardButton("📱 Mobile Lottery (400 Birr)", callback_data="Mobile")]
    ]
    await update.message.reply_text("🎯 Choose lottery category:", reply_markup=InlineKeyboardMarkup(keyboard))
    return CATEGORY

async def category_choice(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    choice = query.data
    context.user_data["category"] = choice

    if choice == "Car":
        context.user_data["item"] = "Lifan 520 Model 2014 Color Silver"
        context.user_data["price"] = 1000
    else:
        context.user_data["item"] = "S25 Ultra 256/12"
        context.user_data["price"] = 400

    keyboard = [
        [InlineKeyboardButton("📱 Telebirr", callback_data="Telebirr")],
        [InlineKeyboardButton("🏦 CBE", callback_data="CBE")]
    ]
    await query.edit_message_text(
        f"🎯 You selected *{context.user_data['item']}* (Price: {context.user_data['price']} Birr)\n\n💰 Choose payment method:",
        parse_mode="Markdown",
        reply_markup=InlineKeyboardMarkup(keyboard)
    )
    return PAYMENT_METHOD

async def payment_choice(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    method = query.data
    context.user_data["payment_method"] = method
    if method == "Telebirr":
        msg = "📱 Pay via Telebirr: 0948309164 (Eyuel)"
    else:
        msg = "🏦 Pay via CBE: 1000439672343 (Eyuel Getye)"
    await query.edit_message_text(f"{msg}\n\n💳 After payment, send your receipt number:")
    return RECEIPT

async def get_receipt(update: Update, context: ContextTypes.DEFAULT_TYPE):
    receipt = update.message.text.strip()
    method = context.user_data["payment_method"]

    if method == "Telebirr" and not receipt.startswith("CL"):
        await update.message.reply_text("❌ Telebirr receipts must start with CL.")
        return RECEIPT
    if method == "CBE" and not receipt.startswith("FT"):
        await update.message.reply_text("❌ CBE receipts must start with FT.")
        return RECEIPT
    if receipt_exists(receipt):
        await update.message.reply_text("❌ This receipt number is already used.")
        return RECEIPT

    context.user_data["receipt"] = receipt
    add_valid_receipt(receipt)
    context.user_data["lotto_number"] = generate_unique_lotto(context.user_data["category"])

    summary = (
        f"📋 *Confirm your details:*\n\n"
        f"👤 Name: {context.user_data['name']}\n"
        f"📱 Phone: {context.user_data['phone']}\n"
        f"🎯 Category: {context.user_data['category']}\n"
        f"📝 Item: {context.user_data['item']}\n"
        f"💰 Price: {context.user_data['price']} Birr\n"
        f"💳 Payment: {context.user_data['payment_method']}\n"
        f"📄 Receipt: {receipt}\n"
        f"🎟️ Lotto: {context.user_data['lotto_number']}"
    )
    keyboard = [
        [InlineKeyboardButton("✅ Confirm", callback_data="confirm"),
         InlineKeyboardButton("❌ Cancel", callback_data="cancel")]
    ]
    await update.message.reply_text(summary, parse_mode="Markdown", reply_markup=InlineKeyboardMarkup(keyboard))
    return CONFIRM

async def confirm_submission(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer()
    if query.data == "cancel":
        await query.edit_message_text("❌ Submission cancelled.")
        return ConversationHandler.END

    user = query.from_user
    cursor.execute("""
        INSERT INTO users (user_id, username, name, phone, category, item, price, payment_method, receipt, lotto)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        user.id, user.username, context.user_data["name"], context.user_data["phone"],
        context.user_data["category"], context.user_data["item"], context.user_data["price"],
        context.user_data["payment_method"], context.user_data["receipt"], context.user_data["lotto_number"]
    ))
    conn.commit()

    await query.edit_message_text(
        f"✅ Payment confirmed!\n🎉 Thank you, {context.user_data['name']}!\n"
        f"Your lottery number: *{context.user_data['lotto_number']}*",
        parse_mode="Markdown"
    )

    return ConversationHandler.END

# ----------------------------
# ADMIN COMMANDS
# ----------------------------
async def addreceipt(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ADMIN_CHAT_ID:
        return await update.message.reply_text("❌ Admin only.")
    if not context.args:
        return await update.message.reply_text("Usage: /addreceipt <number>")
    add_valid_receipt(context.args[0])
    await update.message.reply_text("✅ Receipt added.")

async def logs(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_chat.id != ADMIN_CHAT_ID:
        return await update.message.reply_text("❌ Admin only.")
    cursor.execute("SELECT category, COUNT(*), SUM(price) FROM users GROUP BY category")
    rows = cursor.fetchall()
    msg = "📊 Lottery Stats\n\n"
    for r in rows:
        msg += f"{r[0]}: {r[1]} entries | Revenue: {r[2] or 0} Birr\n"
    await update.message.reply_text(msg)

# ----------------------------
# WINNER SELECTION
# ----------------------------
async def pick_winners(context: ContextTypes.DEFAULT_TYPE):
    for category in ["Car", "Mobile"]:
        cursor.execute("SELECT user_id, username, lotto FROM users WHERE category=? AND winner=0", (category,))
        entries = cursor.fetchall()
        if entries:
            winner = random.choice(entries)
            cursor.execute("UPDATE users SET winner=1 WHERE lotto=?", (winner[2],))
            conn.commit()
            msg = f"🎉 {category} Winner: @{winner[1] or 'N/A'} | {winner[2]}"
            await context.bot.send_message(ADMIN_CHAT_ID, msg)
            await context.bot.send_message(PUBLIC_CHAT_ID, msg)

# ----------------------------
# MAIN (Unified for VS Code + Render)
# ----------------------------
def main():
    logging.basicConfig(level=logging.INFO)
    init_db()

    app = ApplicationBuilder().token(BOT_TOKEN).build()

    conv = ConversationHandler(
        entry_points=[CommandHandler("start", start)],
        states={
            NAME: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_name)],
            PHONE: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_phone)],
            CATEGORY: [CallbackQueryHandler(category_choice)],
            PAYMENT_METHOD: [CallbackQueryHandler(payment_choice)],
            RECEIPT: [MessageHandler(filters.TEXT & ~filters.COMMAND, get_receipt)],
            CONFIRM: [CallbackQueryHandler(confirm_submission)]
        },
        fallbacks=[],
    )

    app.add_handler(conv)
    app.add_handler(CommandHandler("addreceipt", addreceipt))
    app.add_handler(CommandHandler("logs", logs))

    scheduler = AsyncIOScheduler()
    scheduler.add_job(pick_winners, "cron", hour=18, minute=0, args=[app])
    scheduler.start()

    print("[OK] Bot running (Render or local)...")

    async def run():
        await app.initialize()
        await app.start()
        await app.updater.start_polling()
        print("[READY] Bot started successfully!")
        await asyncio.Event().wait()  # Keeps running

    asyncio.run(run())

if __name__ == "__main__":
    main()
