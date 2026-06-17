import os
import json
import logging
from pathlib import Path
from telegram import Update
from telegram.ext import (
    ApplicationBuilder,
    MessageHandler,
    CommandHandler,
    filters,
    ContextTypes,
)

logging.basicConfig(
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    level=logging.INFO
)
logger = logging.getLogger(__name__)

BOT_TOKEN = os.environ["TELEGRAM_BOT_TOKEN"]
OWNER_CHAT_ID = int(os.environ["TELEGRAM_CHAT_ID"])

BOT_DIR = Path(__file__).parent
MAPPING_FILE = BOT_DIR / "message_map.json"
SETTINGS_FILE = BOT_DIR / "settings.json"

DEFAULT_SETTINGS = {
    "auto_reply": "مرحباً! سأرد عليك في أقرب وقت ممكن 😊",
    "welcome_message": "مرحباً! كيف يمكنني مساعدتك؟",
    "silent_mode": False,
    "anonymous_mode": False,
    "blocked_users": [],
}

def load_settings() -> dict:
    if SETTINGS_FILE.exists():
        try:
            data = json.loads(SETTINGS_FILE.read_text())
            return {**DEFAULT_SETTINGS, **data}
        except Exception:
            pass
    return dict(DEFAULT_SETTINGS)

def save_settings(settings: dict) -> None:
    SETTINGS_FILE.write_text(json.dumps(settings, ensure_ascii=False, indent=2))

def load_mapping() -> dict:
    if MAPPING_FILE.exists():
        try:
            return json.loads(MAPPING_FILE.read_text())
        except Exception:
            return {}
    return {}

def save_mapping(mapping: dict) -> None:
    MAPPING_FILE.write_text(json.dumps(mapping))

async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    sender = update.effective_user
    settings = load_settings()
    if sender and sender.id == OWNER_CHAT_ID:
        silent_label = "🔇 صامت" if settings["silent_mode"] else "🔔 عادي"
        anon_label = "🕵️ مفعّل" if settings["anonymous_mode"] else "👤 معطّل"
        await update.message.reply_text(
            f"👋 البوت يعمل!\n\n"
            f"• كل رسالة من المستخدمين ستُرسَل إليك هنا.\n"
            f"• للرد على أي مستخدم، فقط رد على الرسالة المُحوَّلة.\n\n"
            f"الإشعارات: {silent_label}   |   مجهول الهوية: {anon_label}\n\n"
            f"الأوامر المتاحة:\n"
            f"/settings — عرض الإعدادات الحالية\n"
            f"/setreply ‹نص› — تغيير الرد التلقائي\n"
            f"/setwelcome ‹نص› — تغيير رسالة الترحيب\n"
            f"/silent — تبديل الإشعارات (صوت / صامت)\n"
            f"/anonymous — تبديل وضع مجهول الهوية\n"
            f"/block — حظر مستخدم (رد على رسالته)\n"
            f"/unblock — رفع الحظر (رد على رسالته)"
        )
    else:
        await update.message.reply_text(settings["welcome_message"])

async def settings_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    settings = load_settings()
    silent_label = "🔇 صامت" if settings["silent_mode"] else "🔔 عادي"
    anon_label = "🕵️ مفعّل" if settings["anonymous_mode"] else "👤 معطّل"
    blocked = settings["blocked_users"]
    blocked_str = ", ".join(str(u) for u in blocked) if blocked else "لا يوجد"
    await update.message.reply_text(
        f"⚙️ الإعدادات الحالية:\n\n"
        f"📩 الرد التلقائي:\n{settings['auto_reply']}\n\n"
        f"👋 رسالة الترحيب:\n{settings['welcome_message']}\n\n"
        f"🔔 الإشعارات: {silent_label}\n\n"
        f"🕵️ مجهول الهوية: {anon_label}\n\n"
        f"🚫 المحظورون: {blocked_str}"
    )

async def setreply_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if not context.args:
        await update.message.reply_text("الاستخدام: /setreply ‹النص الجديد›")
        return
    new_reply = " ".join(context.args)
    settings = load_settings()
    settings["auto_reply"] = new_reply
    save_settings(settings)
    await update.message.reply_text(f"✅ تم تحديث الرد التلقائي:\n\n{new_reply}")

async def setwelcome_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if not context.args:
        await update.message.reply_text("الاستخدام: /setwelcome ‹النص الجديد›")
        return
    new_welcome = " ".join(context.args)
    settings = load_settings()
    settings["welcome_message"] = new_welcome
    save_settings(settings)
    await update.message.reply_text(f"✅ تم تحديث رسالة الترحيب:\n\n{new_welcome}")

async def silent_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    settings = load_settings()
    settings["silent_mode"] = not settings["silent_mode"]
    save_settings(settings)
    if settings["silent_mode"]:
        await update.message.reply_text("🔇 تم التفعيل — ستصلك الرسائل بدون صوت.")
    else:
        await update.message.reply_text("🔔 تم الإلغاء — ستصلك الرسائل مع صوت.")

async def anonymous_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    settings = load_settings()
    settings["anonymous_mode"] = not settings["anonymous_mode"]
    save_settings(settings)
    if settings["anonymous_mode"]:
        await update.message.reply_text("🕵️ تم تفعيل وضع مجهول الهوية.")
    else:
        await update.message.reply_text("👤 تم إلغاء وضع مجهول الهوية.")

async def block_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    message = update.effective_message
    if not message or not message.reply_to_message:
        await message.reply_text("⚠️ يجب أن ترد على رسالة المستخدم الذي تريد حظره.")
        return
    replied_msg_id = str(message.reply_to_message.message_id)
    mapping = load_mapping()
    if replied_msg_id not in mapping:
        await message.reply_text("⚠️ لا يمكن تحديد المستخدم من هذه الرسالة.")
        return
    original_chat_id = mapping[replied_msg_id]
    settings = load_settings()
    if original_chat_id in settings["blocked_users"]:
        await message.reply_text("ℹ️ هذا المستخدم محظور مسبقاً.")
        return
    settings["blocked_users"].append(original_chat_id)
    save_settings(settings)
    await message.reply_text("🚫 تم حظر المستخدم.")

async def unblock_command(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    message = update.effective_message
    if not message or not message.reply_to_message:
        await message.reply_text("⚠️ يجب أن ترد على رسالة المستخدم الذي تريد رفع الحظر عنه.")
        return
    replied_msg_id = str(message.reply_to_message.message_id)
    mapping = load_mapping()
    if replied_msg_id not in mapping:
        await message.reply_text("⚠️ لا يمكن تحديد المستخدم من هذه الرسالة.")
        return
    original_chat_id = mapping[replied_msg_id]
    settings = load_settings()
    if original_chat_id not in settings["blocked_users"]:
        await message.reply_text("ℹ️ هذا المستخدم غير محظور.")
        return
    settings["blocked_users"].remove(original_chat_id)
    save_settings(settings)
    await message.reply_text("✅ تم رفع الحظر.")

async def handle_user_message(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    message = update.effective_message
    sender = update.effective_user
    chat = update.effective_chat
    if not message or not chat or not sender:
        return
    settings = load_settings()
    if sender.id in settings["blocked_users"]:
        return
    await message.reply_text(settings["auto_reply"])
    silent = settings["silent_mode"]
    anon = settings["anonymous_mode"]
    try:
        if anon:
            sent = await context.bot.send_message(
                chat_id=OWNER_CHAT_ID,
                text=f"📨 رسالة جديدة (مجهول):\n\n{message.text or '[محتوى]'}",
                disable_notification=silent,
            )
            sent_id = sent.message_id
        else:
            forwarded = await context.bot.forward_message(
                chat_id=OWNER_CHAT_ID,
                from_chat_id=chat.id,
                message_id=message.message_id,
                disable_notification=silent,
            )
            sent_id = forwarded.message_id
        if sent_id:
            mapping = load_mapping()
            mapping[str(sent_id)] = chat.id
            save_mapping(mapping)
    except Exception as e:
        logger.error("Failed to forward message: %s", e)

async def handle_owner_reply(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    message = update.effective_message
    if not message or not message.reply_to_message:
        return
    replied_msg_id = str(message.reply_to_message.message_id)
    mapping = load_mapping()
    if replied_msg_id not in mapping:
        await message.reply_text("⚠️ لا يمكن تحديد المرسل الأصلي.")
        return
    original_chat_id = mapping[replied_msg_id]
    try:
        if message.text:
            await context.bot.send_message(chat_id=original_chat_id, text=message.text)
        await message.reply_text("✅ تم إرسال ردك.")
    except Exception as e:
        await message.reply_text(f"❌ فشل الإرسال: {e}")

def main() -> None:
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    owner_filter = filters.Chat(chat_id=OWNER_CHAT_ID)
    user_filter = ~owner_filter
    app.add_handler(CommandHandler("start", start_command))
    app.add_handler(CommandHandler("settings", settings_command, filters=owner_filter))
    app.add_handler(CommandHandler("setreply", setreply_command, filters=owner_filter))
    app.add_handler(CommandHandler("setwelcome", setwelcome_command, filters=owner_filter))
    app.add_handler(CommandHandler("silent", silent_command, filters=owner_filter))
    app.add_handler(CommandHandler("anonymous", anonymous_command, filters=owner_filter))
    app.add_handler(CommandHandler("block", block_command, filters=owner_filter))
    app.add_handler(CommandHandler("unblock", unblock_command, filters=owner_filter))
    app.add_handler(MessageHandler(owner_filter & filters.REPLY & ~filters.COMMAND, handle_owner_reply))
    app.add_handler(MessageHandler(user_filter & ~filters.COMMAND, handle_user_message))
    logger.info("Bot started.")
    app.run_polling(allowed_updates=Update.ALL_TYPES)

if __name__ == "__main__":
    main()
