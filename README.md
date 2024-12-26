import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton, Message
import os
import json
import random
import time

# --- Load bot token from environment variables ---
BOT_TOKEN = os.getenv("BOT_TOKEN")
if not BOT_TOKEN:
    raise ValueError("Please set the BOT_TOKEN environment variable.")

bot = telebot.TeleBot(BOT_TOKEN)

# --- Paths for saving data ---
USER_DATA_FILE = "/tmp/user_data.json"  # تغییر مسیر ذخیره فایل به /tmp
MESSAGE_DATA_FILE = "/tmp/message_data.json"  # تغییر مسیر ذخیره فایل به /tmp
CHANNELS = ["@khclick", "@khclick_addlist", "@dastyarchannelnews"]

# --- Load or initialize data ---
def load_data(file_name):
    try:
        with open(file_name, "r") as file:
            return json.load(file)
    except FileNotFoundError:
        return {}

def save_data(file_name, data):
    with open(file_name, "w") as file:
        json.dump(data, file, ensure_ascii=False, indent=4)

user_data = load_data(USER_DATA_FILE)

# --- Utility functions ---
def check_joined_channels(user_id):
    for channel in CHANNELS:
        try:
            status = bot.get_chat_member(channel, user_id).status
            if status not in ["member", "administrator", "creator"]:
                return False
        except Exception:
            return False
    return True

def generate_random_coins():
    return random.randint(50, 70)

# --- Commands ---
@bot.message_handler(commands=["start"])
def start_handler(message: Message):
    user_id = str(message.from_user.id)
    if user_id not in user_data:
        user_data[user_id] = {
            "coins": 0,
            "invited_by": None,
            "daily_check": "",
            "joined": False,
        }
        save_data(USER_DATA_FILE, user_data)
    if not check_joined_channels(message.from_user.id):
        markup = InlineKeyboardMarkup()
        for channel in CHANNELS:
            markup.add(InlineKeyboardButton("Join", url=f"https://t.me/{channel[1:]}"))
        bot.send_message(
            message.chat.id,
            "برای استفاده از ربات، ابتدا در کانال‌های زیر عضو شوید:",
            reply_markup=markup,
        )
    else:
        user = user_data[user_id]
        if not user["joined"]:
            coins = generate_random_coins()
            user["coins"] += coins
            user["joined"] = True
            save_data(USER_DATA_FILE, user_data)
            bot.send_message(
                message.chat.id,
                f"به ربات خوش آمدید! {coins} سکه به شما داده شد.\nاز دستورات زیر استفاده کنید:",
            )
        else:
            bot.send_message(
                message.chat.id,
                "شما قبلاً وارد ربات شده‌اید! از دستورات زیر استفاده کنید:",
            )
    bot.send_message(
        message.chat.id,
        "/help - دریافت راهنما\n"
        "/profile - مشاهده پروفایل\n"
        "/create_button - ساخت دکمه شیشه‌ای\n"
        "/invite - دعوت دوستان و دریافت سکه\n"
        "/check_daily - دریافت سکه روزانه\n"
        "/subscriptions - خرید اشتراک\n"
    )

@bot.message_handler(commands=["help"])
def help_handler(message: Message):
    bot.send_message(
        message.chat.id,
        "راهنمای دستورات:\n"
        "/profile - مشاهده پروفایل و میزان سکه‌ها\n"
        "/invite - دعوت دوستان و دریافت سکه\n"
        "/check_daily - دریافت سکه روزانه\n"
        "/subscriptions - خرید اشتراک\n"
        "برای هر گونه سوال می‌توانید از طریق پشتیبانی اقدام کنید.",
    )

@bot.message_handler(commands=["profile"])
def profile_handler(message: Message):
    user_id = str(message.from_user.id)
    user = user_data.get(user_id, {})
    coins = user.get("coins", 0)
    invites = len([u for u in user_data.values() if u.get("invited_by") == user_id])
    bot.send_message(
        message.chat.id,
        f"پروفایل شما:\n"
        f"سکه‌ها: {coins}\n"
        f"دعوت‌ها: {invites}",
    )

@bot.message_handler(commands=["invite"])
def invite_handler(message: Message):
    user_id = str(message.from_user.id)
    bot.send_message(
        message.chat.id,
        f"لینک دعوت شما:\nhttps://t.me/{bot.get_me().username}?start={user_id}\n"
        "هر دعوت موفقیت‌آمیز:\n- شما: 10 سکه\n- کاربر جدید: 5 سکه",
    )

@bot.message_handler(commands=["check_daily"])
def check_daily_handler(message: Message):
    user_id = str(message.from_user.id)
    user = user_data.get(user_id, {})
    today = time.strftime("%Y-%m-%d")
    if user.get("daily_check") == today:
        bot.send_message(message.chat.id, "شما قبلاً امروز سکه دریافت کرده‌اید!")
        return
    coins = 10
    user["coins"] += coins
    user["daily_check"] = today
    save_data(USER_DATA_FILE, user_data)
    bot.send_message(
        message.chat.id,
        f"شما {coins} سکه برای امروز دریافت کردید!\n"
        "فردا دوباره به ربات سر بزنید.",
    )

@bot.message_handler(commands=["subscriptions"])
def subscriptions_handler(message: Message):
    bot.send_message(
        message.chat.id,
        "اشتراک‌های موجود:\n"
        "1. 1 ماهه - 100 سکه\n"
        "2. 3 ماهه - 250 سکه\n"
        "3. 6 ماهه - 450 سکه\n"
        "برای خرید، کد اشتراک را ارسال کنید (مثلاً: 1 یا 2 یا 3).",
    )

# --- Fallback handler for unknown messages ---
@bot.message_handler(func=lambda message: True)
def fallback_handler(message: Message):
    bot.send_message(
        message.chat.id,
        "متوجه دستور شما نشدم! لطفاً از /help استفاده کنید.",
    )

# --- Start the bot ---
if __name__ == "__main__":
    save_data(USER_DATA_FILE, user_data)
    bot.infinity_polling(timeout=10, long_polling_timeout=5)