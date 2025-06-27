# Nex-Uploader-
Telegram uploader bot
<xaiArtifact artifact_id="1aef1bb4-9489-49ba-9829-f4c2234403c8" artifact_version_id="8d003e7e-3cff-4bb4-84ef-61030c8a0ea4" title="bot.py" contentType="text/python">
from telegram.ext import Application, CommandHandler, MessageHandler, Filters, ContextTypes
from telegram import Update
import os
import json
import uuid
import logging

# Environment variables
TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")
CHANNEL_ID = "@nexvpn_server"
ADMIN_ID = os.getenv("ADMIN_ID")  # Your Telegram user ID

# Setup logging
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# File to store file_id and link mappings
DATA_FILE = "files.json"

# Load or initialize the file database
def load_files():
    try:
        with open(DATA_FILE, 'r') as f:
            return json.load(f)
    except FileNotFoundError:
        return {}

def save_files(files):
    with open(DATA_FILE, 'w') as f:
        json.dump(files, f, indent=4)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    await context.bot.send_message(chat_id, "خوش اومدی! برای دانلود فایل، روی لینکی که ادمین می‌ده کلیک کن. اول مطمئن شو تو کانال @nexvpn_server جوین شدی!")

async def handle_file(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = update.effective_user.id
    chat_id = update.effective_chat.id
    
    # Check if user is admin
    if str(user_id) != ADMIN_ID:
        await context.bot.send_message(chat_id, "فقط ادمین می‌تونه فایل آپلود کنه!")
        return

    if update.message.document or update.message.video:
        file = update.message.document or update.message.video
        file_id = file.file_id
        file_name = file.file_name if file.file_name else "فایل بدون نام"
        
        # Generate unique link ID
        link_id = str(uuid.uuid4())
        files = load_files()
        files[link_id] = {"file_id": file_id, "file_name": file_name}
        save_files(files)
        
        # Create download link (using bot username and link_id)
        bot_username = (await context.bot.get_me()).username
        download_link = f"https://t.me/{bot_username}?start=download_{link_id}"
        
        await context.bot.send_message(chat_id, f"فایل آپلود شد: {file_name}\nلینک دانلود: `{download_link}`\nاین لینک رو برای کاربرها بفرست!")
    else:
        await context.bot.send_message(chat_id, "لطفاً یه فایل یا فیلم آپلود کن!")

async def handle_download(update: Update, context: ContextTypes.DEFAULT_TYPE):
    chat_id = update.effective_chat.id
    user_id = update.effective_user.id
    
    # Check if user is in the channel
    try:
        member = await context.bot.get_chat_member(CHANNEL_ID, user_id)
        if member.status not in ["member", "administrator", "creator"]:
            await context.bot.send_message(chat_id, f"لطفاً اول به {CHANNEL_ID} جوین شو!")
            return
    except Exception as e:
        logger.error(f"Error checking channel membership: {e}")
        await context.bot.send_message(chat_id, "خطایی پیش اومد! مطمئن شو که ربات ادمین کاناله.")
        return

    # Extract link_id from /start download_xxx
    if not context.args or not context.args[0].startswith("download_"):
        await context.bot.send_message(chat_id, "لینک دانلود معتبر نیست!")
        return

    link_id = context.args[0].replace("download_", "")
    files = load_files()
    
    if link_id not in files:
        await context.bot.send_message(chat_id, "فایل پیدا نشد! لینک اشتباه است
