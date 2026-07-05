"""
PowerAlert GH - Crowdsourced power outage tracker for Ghana
Built with python-telegram-bot
"""

import sqlite3
import logging
from datetime import datetime, timedelta

from telegram import Update
from telegram.ext import (
    Application,
    CommandHandler,
    MessageHandler,
    ContextTypes,
    ConversationHandler,
    filters,
)

import os
BOT_TOKEN = "8093500509:AAFBbSe_AkWGXI6Bgp2jB6cv0WoDSu7Awk0"
DB_PATH = "poweralert.db"

import os
BOT_TOKEN = os.environ.get("BOT_TOKEN")
ADMIN_ID = 8547003210
DB_PATH = "poweralert.db"


AWAITING_AREA_REPORT, AWAITING_AREA_STATUS, AWAITING_AREA_RESTORED, AWAITING_AREA_SUBSCRIBE = range(4)

logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO
)
logger = logging.getLogger(__name__)


def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS outage_reports (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            username TEXT,
            area TEXT,
            status TEXT,
            timestamp TEXT
        )
    """)
    c.execute("""
        CREATE TABLE IF NOT EXISTS subscriptions (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER,
            area TEXT
        )
    """)
    conn.commit()
    conn.close()


def add_report(user_id, username, area, status):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute(
        "INSERT INTO outage_reports (user_id, username, area, status, timestamp) VALUES (?, ?, ?, ?, ?)",
        (user_id, username, area.strip().lower(), status, datetime.now().isoformat())
    )
    conn.commit()
    conn.close()


def get_recent_reports(area, hours=6):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    cutoff = (datetime.now() - timedelta(hours=hours)).isoformat()
    c.execute(
        "SELECT username, status, timestamp FROM outage_reports WHERE area = ? AND timestamp > ? ORDER BY timestamp DESC",
        (area.strip().lower(), cutoff)
    )
    rows = c.fetchall()
    conn.close()
    return rows


def add_subscription(user_id, area):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("INSERT INTO subscriptions (user_id, area) VALUES (?, ?)", (user_id, area.strip().lower()))
    conn.commit()
    conn.close()


def get_subscribers(area):
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("SELECT user_id FROM subscriptions WHERE area = ?", (area.strip().lower(),))
    rows = c.fetchall()
    conn.close()
    return [r[0] for r in rows]


async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text(
        "Welcome to PowerAlert GH!\n\n"
        "Crowdsourced power outage tracking for Ghana.\n\n"
        "Commands:\n"
        "/report - Report an outage in your area\n"
        "/status - Check recent reports for an area\n"
        "/restored - Mark power as restored\n"
        "/subscribe - Get notified about outages near you\n"
        "/help - Show this again"
    )


async def help_cmd(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await start(update, context)


async def report_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Which area is the outage in? (e.g. East Legon, Adenta, Kasoa)")
    return AWAITING_AREA_REPORT


async def report_area(update: Update, context: ContextTypes.DEFAULT_TYPE):
    area = update.message.text
    user = update.effective_user
    add_report(user.id, user.username or user.first_name, area, "outage")

    subscribers = get_subscribers(area)
    for sub_id in subscribers:
        if sub_id != user.id:
            try:
                await context.bot.send_message(
                    sub_id, "New outage reported in " + area.strip().title()
                )
            except Exception as e:
                logger.warning("Could not notify " + str(sub_id) + ": " + str(e))
    try:
        reporter_name = user.username or user.first_name or "Unknown"
        await context.bot.send_message(
            ADMIN_ID,
            "ADMIN ALERT\nNew outage report\nArea: " + area.strip().title() + 
            "\nReported by: " + reporter_name + 
            "\nTime: " + datetime.now().strftime("%Y-%m-%d %H:%M")
        )
    except Exception as e:
        logger.warning("Could not notify admin: " + str(e))
    await update.message.reply_text("Outage logged for " + area.strip().title() + ". Thanks for reporting!")

    return ConversationHandler.END


async def status_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Which area do you want to check?")
    return AWAITING_AREA_STATUS


async def status_area(update: Update, context: ContextTypes.DEFAULT_TYPE):
    area = update.message.text
    reports = get_recent_reports(area)

    if not reports:
        await update.message.reply_text(
            "No reports for " + area.strip().title() + " in the last 6 hours. "
            "Could just be you, or all clear!"
        )
    else:
        lines = ["Recent reports for " + area.strip().title() + ":"]
        for username, status, ts in reports[:10]:
            marker = "[OUTAGE]" if status == "outage" else "[RESTORED]"
            time_str = datetime.fromisoformat(ts).strftime("%H:%M")
            lines.append(marker + " " + status.title() + " - " + time_str + " (by " + (username or "anon") + ")")
        await update.message.reply_text("\n".join(lines))

    return ConversationHandler.END


async def restored_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Which area has power back on?")
    return AWAITING_AREA_RESTORED


async def restored_area(update: Update, context: ContextTypes.DEFAULT_TYPE):
    area = update.message.text
    user = update.effective_user
    add_report(user.id, user.username or user.first_name, area, "restored")

    try:
        reporter_name = user.username or user.first_name or "Unknown"
        await context.bot.send_message(
            ADMIN_ID,
            "ADMIN ALERT\nPower restored\nArea: " + area.strip().title() + 
            "\nReported by: " + reporter_name + 
            "\nTime: " + datetime.now().strftime("%Y-%m-%d %H:%M")
        )
    except Exception as e:
        logger.warning("Could not notify admin: " + str(e))
        
    await update.message.reply_text("Marked as restored for " + area.strip().title() + ". Good news!")
    return ConversationHandler.END


async def subscribe_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Which area do you want alerts for?")
    return AWAITING_AREA_SUBSCRIBE


async def subscribe_area(update: Update, context: ContextTypes.DEFAULT_TYPE):
    area = update.message.text
    user = update.effective_user
    add_subscription(user.id, area)
    await update.message.reply_text("You'll be notified about outages in " + area.strip().title() + ".")
    return ConversationHandler.END


async def cancel(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("Cancelled.")
    return ConversationHandler.END


def main():
    init_db()
    app = Application.builder().token(BOT_TOKEN).build()

    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("help", help_cmd))

    report_conv = ConversationHandler(
        entry_points=[CommandHandler("report", report_start)],
        states={AWAITING_AREA_REPORT: [MessageHandler(filters.TEXT & ~filters.COMMAND, report_area)]},
        fallbacks=[CommandHandler("cancel", cancel)],
    )
    status_conv = ConversationHandler(
        entry_points=[CommandHandler("status", status_start)],
        states={AWAITING_AREA_STATUS: [MessageHandler(filters.TEXT & ~filters.COMMAND, status_area)]},
        fallbacks=[CommandHandler("cancel", cancel)],
    )
    restored_conv = ConversationHandler(
        entry_points=[CommandHandler("restored", restored_start)],
        states={AWAITING_AREA_RESTORED: [MessageHandler(filters.TEXT & ~filters.COMMAND, restored_area)]},
        fallbacks=[CommandHandler("cancel", cancel)],
    )
    subscribe_conv = ConversationHandler(
        entry_points=[CommandHandler("subscribe", subscribe_start)],
        states={AWAITING_AREA_SUBSCRIBE: [MessageHandler(filters.TEXT & ~filters.COMMAND, subscribe_area)]},
        fallbacks=[CommandHandler("cancel", cancel)],
    )

    app.add_handler(report_conv)
    app.add_handler(status_conv)
    app.add_handler(restored_conv)
    app.add_handler(subscribe_conv)

    logger.info("PowerAlert GH bot starting...")
    app.run_polling()


if __name__ == "__main__":
    main()
    