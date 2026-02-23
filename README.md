# Telegram-bot
bot.py
import telebot
from telebot import types
import json
import os
from datetime import datetime

TOKEN = "8582313463:AAEWSz_u0GdGV0IxUKSRwLoZGC86yFb_3_A"
ADMIN_ID = 8501119749
CHANNELS = ["@moneey_uz", "@zayfka4738"]
DATA_FILE = "users.json"

bot = telebot.TeleBot(TOKEN)

def load_data():
    if os.path.exists(DATA_FILE):
        with open(DATA_FILE, 'r') as f:
            return json.load(f)
    return {}

def save_data(data):
    with open(DATA_FILE, 'w') as f:
        json.dump(data, f)

users_data = load_data()

def main_menu():
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    markup.add(types.KeyboardButton("📊 Shaxsiy kabinet"))
    markup.add(types.KeyboardButton("💰 Chiqarish va statistika"))
    markup.add(types.KeyboardButton("📞 Support"))
    return markup

def admin_menu():
    markup = types.InlineKeyboardMarkup()
    markup.add(types.InlineKeyboardButton("👥 Foydalanuvchilar", callback_data="admin_users"))
    markup.add(types.InlineKeyboardButton("📤 Broadcast", callback_data="admin_broadcast"))
    markup.add(types.InlineKeyboardButton("📢 Reklama qo'shish", callback_data="admin_ads"))
    markup.add(types.InlineKeyboardButton("💸 Chiqishlar", callback_data="admin_withdraw"))
    return markup

@bot.message_handler(commands=['start'])
def start(message):
    user_id = str(message.from_user.id)
    
    if user_id in users_data:
        bot.send_message(message.chat.id, "👋 Siz avval bu botda bo'lgansiz!", reply_markup=main_menu())
        return
    
    users_data[user_id] = {
        "name": message.from_user.first_name,
        "stars": 0,
        "referrer": None,
        "referrals": [],
        "subscribed": False,
        "join_date": datetime.now().strftime("%Y-%m-%d %H:%M")
    }
    
    if len(message.text.split()) > 1:
        ref_code = message.text.split()[1]
        for uid, data in users_data.items():
            if data.get("ref_code") == ref_code and uid != user_id:
                users_data[user_id]["referrer"] = uid
                users_data[uid]["stars"] += 2
                users_data[uid]["referrals"].append(user_id)
                bot.send_message(uid, f"Yangi referral! +2 stars (Jami: {users_data[uid]['stars']})")
                break
    
    users_data[user_id]["ref_code"] = f"ref_{user_id[:8]}"
    
    markup = types.InlineKeyboardMarkup()
    for channel in CHANNELS:
        markup.add(types.InlineKeyboardButton(f"Obuna bo'lish {channel}", url=f"https://t.me/{channel[1:]}"))
    markup.add(types.InlineKeyboardButton("✅ Tekshirish", callback_data="check_subs"))
    
    welcome_text = f"🎉 Xush kelibsiz, {message.from_user.first_name}!

📢 Majburiy kanallarga obuna bo'ling va 'Tekshirish' tugmasini bosing.

🔗 Sizning referralingiz: t.me/{bot.get_me().username}?start={users_data[user_id]['ref_code']}"
    bot.send_message(message.chat.id, welcome_text, reply_markup=markup)
    
    save_data(users_data)

@bot.message_handler(func=lambda message: True)
def handle_menu(message):
    user_id = str(message.from_user.id)
    if user_id not in users_data:
        bot.send_message(message.chat.id, "Iltimos /start bosing", reply_markup=main_menu())
        return
    
    if message.text == "📊 Shaxsiy kabinet":
        stars = users_data[user_id]["stars"]
        referrals = len(users_data[user_id]["referrals"])
        ref_link = f"t.me/{bot.get_me().username}?start={users_data[user_id]['ref_code']}"
        text = f"📊 Shaxsiy kabinet

⭐ Stars: {stars}/100
👥 Referrallar: {referrals}
🔗 Referral link: {ref_link}
📅 Ro'yxat: {users_data[user_id]['join_date']}"
        bot.send_message(message.chat.id, text, reply_markup=main_menu())
    
    elif message.text == "💰 Chiqarish va statistika":
        stars = users_data[user_id]["stars"]
        if stars >= 100:
            markup = types.InlineKeyboardMarkup()
            markup.add(types.InlineKeyboardButton("💸 Stars yechib olish", callback_data="withdraw"))
            bot.send_message(message.chat.id, f"⭐ Sizning balansingiz: {stars}

Yechib olish uchun tugmani bosing", reply_markup=markup)
        else:
            bot.send_message(message.chat.id, f"⭐ Balans: {stars}/100

100 stars to'plang!", reply_markup=main_menu())
    
    elif message.text == "📞 Support":
        bot.send_message(message.chat.id, "📞 Support: @sizning_support

Savollaringiz bo'lsa yozing!", reply_markup=main_menu())

@bot.callback_query_handler(func=lambda call: True)
def callback(call):
    user_id = str(call.from_user.id)
    
    if call.data == "check_subs":
        users_data[user_id]["subscribed"] = True
        bot.edit_message_text("✅ Obuna tasdiqlandi! Endi barcha funksiyalardan foydalaning.", 
                            call.message.chat.id, call.message.id, reply_markup=main_menu())
    
    elif call.data == "withdraw":
        if users_data[user_id]["stars"] >= 100:
            bot.send_message(ADMIN_ID, f"💰 Yechib olish so'rovi
User: {users_data[user_id]['name']}
Stars: {users_data[user_id]['stars']}
ID: {user_id}")
            bot.answer_callback_query(call.id, "✅ So'rov admin tasdiqlashini kutmoqda")
        else:
            bot.answer_callback_query(call.id, "❌ 100 stars yetarli emas")
    
    elif str(call.from_user.id) == str(ADMIN_ID):
        if call.data == "admin_panel":
            bot.edit_message_text("🔧 Admin panel", call.message.chat.id, call.message.id, reply_markup=admin_menu())
        
        elif call.data == "admin_users":
            text = "👥 Foydalanuvchilar:

"
            for uid, data in users_data.items():
                text += f"• {data['name']} (ID: {uid}) - {data['stars']}⭐
"
            bot.edit_message_text(text[:4000], call.message.chat.id, call.message.id)
        
        elif call.data == "admin_broadcast":
            bot.send_message(call.message.chat.id, "📤 Broadcast xabar yuboring:")
            bot.register_next_step_handler(call.message, broadcast_message)
        
        elif call.data == "admin_ads":
            bot.send_message(call.message.chat.id, "📢 Yan2gi reklama kanal yuboring (@username):")
            bot.register_next_step_handler(call.message, add_ad_channel)

def broadcast_message(message):
    if str(message.from_user.id) == str(ADMIN_ID):
        for uid in users_data:
            try:
                bot.send_message(uid, message.text)
            except:
                pass
        bot.send_message(ADMIN_ID, "✅ Broadcast jo'natildi!")

def add_ad_channel(message):
    global CHANNELS
    if str(message.from_user.id) == str(ADMIN_ID):
        CHANNELS.append(message.text)
        bot.send_message(ADMIN_ID, f"✅ Kanal qo'shildi: {message.text}")

@bot.message_handler(commands=['admin'])
def admin_start(message):
    if str(message.from_user.id) == str(ADMIN_ID):
        markup = types.InlineKeyboardMarkup()
        markup.add(types.InlineKeyboardButton("🔧 Admin panel", callback_data="admin_panel"))
        bot.send_message(message.chat.id, "🔧 Admin panel ochildi", reply_markup=markup)

if __name__ == "__main__":
    bot.polling(none_stop=True)
