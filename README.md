# Ato-Movie
import os
import time
import requests
from pymongo import MongoClient
from pyrogram import Client, filters
from pyrogram.types import InlineKeyboardMarkup, InlineKeyboardButton

API_ID = int(os.getenv("22949670"))  
API_HASH = os.getenv("cef9ce2e0ddcca5188a362802e86fd18")
BOT_TOKEN = os.getenv("7766714140:AAFZPo9Nr183mrrH2H_frWNd-V7WgxSKnZA")
FORCE_SUB_CHANNEL = os.getenv("https://t.me/moviestudio444")
MONGO_URI = os.getenv("mongodb+srv://atanusikdar2:MOVIESTUDIO2006@moviestudio.j78xo.mongodb.net/?retryWrites=true&w=majority&appName=Moviestudio")
LOG_CHANNEL = os.getenv("https://t.me/+YkjhiiuiQb40OWE1")
ADMIN_ID = int(os.getenv("5558799839"))
SHORTENER_API = os.getenv("88184deffcb7348f0aabf0468eb14516941fc646")
SHORTENER_URL = os.getenv("SHORTENER_URL", "https://omegalinks.in/st?api=88184deffcb7348f0aabf0468eb14516941fc646&url=yourdestinationlink.com")
CHANNEL_USERNAME = os.getenv("CHANNEL_USERNAME", "https://t.me/moviestudio444")
START_IMAGE = os.getenv("START_IMAGE", "https://graph.org/file/76bc727099c8755565cbb-ba705eb10e6aabd3e9.jpg")
START_TEXT = """✨ Welcome to Auto Filter Bot! ✨

🔍 Search Movies & Files Easily
📥 Just send the name and get the results instantly!
📢 Join our channel for updates!

⚡ Enjoy hassle-free searching! ⚡"""

bot = Client("AutoFilterBot", api_id=API_ID, api_hash=API_HASH, bot_token=BOT_TOKEN)

db_client = MongoClient(MONGO_URI)
db = db_client['AutoFilterDB']
files_collection = db['files']
users_collection = db['users']
stats_collection = db['stats']

def shorten_url(url):
    try:
        response = requests.get(f"{SHORTENER_URL}?api={SHORTENER_API}&url={url}")
        data = response.json()
        if data["status"] == "success":
            return data["shortenedUrl"]
    except Exception as e:
        print(f"Error shortening URL: {e}")
    return url

def check_subscription(user_id):
    try:
        member = bot.get_chat_member(FORCE_SUB_CHANNEL, user_id)
        return member.status in ["member", "administrator", "creator"]
    except:
        return False

@bot.on_message(filters.private & filters.command("start"))
def start(client, message):
    user_id = message.from_user.id
    if not check_subscription(user_id):
        keyboard = InlineKeyboardMarkup([
            [InlineKeyboardButton("🚀 Join Channel", url=f"https://t.me/moviestudio444")]
        ])
        return message.reply_text("⚠ Please join our channel to use this bot!", reply_markup=keyboard)
    
    users_collection.update_one({"user_id": user_id}, {"$set": {"user_id": user_id}}, upsert=True)
    message.reply_chat_action("typing")
    time.sleep(1)
    message.reply_photo(photo=START_IMAGE, caption=START_TEXT)

@bot.on_message(filters.text & filters.private)
def search_files(client, message):
    query = message.text
    stats_collection.update_one({"query": query}, {"$inc": {"count": 1}}, upsert=True)
    
    files = files_collection.find({"file_name": {"$regex": query, "$options": "i"}})
    results = []
    for f in files:
        short_url = shorten_url(f["file_url"])
        caption = f"🎬 {f['file_name']}\n🔗 🎥 Watch Now\n📢 Join {CHANNEL_USERNAME} for more!"
        results.append([InlineKeyboardButton(f"📥 {f['file_name']}", url=short_url)])

    if results:
        keyboard = InlineKeyboardMarkup(results)
        message.reply_chat_action("typing")
        time.sleep(1)
        message.reply_text(caption, disable_web_page_preview=True, reply_markup=keyboard)
    else:
        message.reply_text("❌ No files found. Try again!")

@bot.on_message(filters.command("help"))
def help_command(client, message):
    help_text = """🆘 Help Menu 🆘

🔍 How to Use?
1️⃣ Send the movie or file name.
2️⃣ Get instant results with links.

⚙ Admin Commands:
- /stats - Show bot statistics.
- /broadcast <message> - Send a message to all users.
- /ban <user_id> - Ban a user.
- /unban <user_id> - Unban a user.
- /log - Log user activity.
- /users - Show total registered users.
- /topsearches - Show top searched queries.

📢 Join our channel for updates!"""
    
    keyboard = InlineKeyboardMarkup([
        [InlineKeyboardButton("📢 Join Channel", url=f"https://t.me/moviestudio444")]
    ])
    message.reply_text(help_text, reply_markup=keyboard)

@bot.on_message(filters.command("users"))
def total_users(client, message):
    count = users_collection.count_documents({})
    message.reply_text(f"👥 Total Users: {count}")

@bot.on_message(filters.command("topsearches"))
def top_searches(client, message):
    top = stats_collection.find().sort("count", -1).limit(5)
    text = "🔥 Top Searches:\n\n"
    for s in top:
        text += f"🔍 {s['query']} - {s['count']} searches\n"
    message.reply_text(text)

if name == "main":
    bot.run()
