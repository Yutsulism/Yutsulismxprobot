# Yutsulismxprobot
import random
from datetime import datetime, timedelta
from telegram import Update, Poll, ParseMode, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Updater, CommandHandler, PollHandler, CallbackQueryHandler, CallbackContext
from pymongo import MongoClient

# Replace these with your actual credentials
BOT_TOKEN = 'YOUR_BOT_TOKEN'
API_ID = 'YOUR_API_ID'
API_HASH = 'YOUR_API_HASH'
MONGO_URI = 'YOUR_MONGO_URI'

# Initialize MongoDB client
client = MongoClient(MONGO_URI)
db = client['telegram_bot']
users = db['users']
referrals = db['referrals']
withdrawals = db['withdrawals']

# Initialize updater and dispatcher
updater = Updater(BOT_TOKEN, use_context=True)
dispatcher = updater.dispatcher

# Helper function to check if user is admin
def is_admin(user_id):
    return users.find_one({'user_id': user_id, 'is_admin': True}) is not None

# Helper function to check if user is the owner
def is_owner(user_id):
    return users.find_one({'user_id': user_id, 'is_owner': True}) is not None

# Welcome message handler
def start(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    user_name = update.message.from_user.username
    user = users.find_one({'user_id': user_id})
    
    if not user:
        users.insert_one({'user_id': user_id, 'username': user_name, 'radiance': 0, 'is_admin': False, 'is_owner': False})
    
    # Owner check
    if is_owner(user_id):
        context.bot.send_message(chat_id=update.effective_chat.id, text=f"Hello God, welcome back!")
    elif is_admin(user_id):
        context.bot.send_message(chat_id=update.effective_chat.id, text=f"Hello Admin {user_name}, welcome back!")
    else:
        context.bot.send_message(chat_id=update.effective_chat.id, text="Welcome! Please join the required channels to proceed.")

    # Inline buttons for channel joining
    keyboard = [
        [InlineKeyboardButton("Channel 1", url="https://t.me/joinchat/XXXXXX"), InlineKeyboardButton("Channel 2", url="https://t.me/joinchat/YYYYYY")],
        [InlineKeyboardButton("Channel 3", url="https://t.me/joinchat/ZZZZZZ"), InlineKeyboardButton("Channel 4", url="https://t.me/joinchat/AAAAAA")],
        [InlineKeyboardButton("I've joined all channels", callback_data='joined')]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    update.message.reply_text('Join the channels below and click "I\'ve joined all channels" when done.', reply_markup=reply_markup)

def button(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    user_id = query.from_user.id

    if query.data == 'joined':
        context.bot.send_message(chat_id=query.message.chat_id, text="Welcome to the Main Menu!\nOptions:\n/withdraw\n/refer\n/balance")

# Generate referral link
def refer(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    referral_link = f"https://t.me/yourbotname?start={user_id}"
    update.message.reply_text(f"Refer and get Radiance and claim free Telegram Premium!\nYour referral link: {referral_link}")

# Handle new user joining via referral link
def handle_new_user(update: Update, context: CallbackContext) -> None:
    args = context.args
    if len(args) > 0:
        referrer_id = int(args[0])
        referred_id = update.message.from_user.id
        if referrer_id != referred_id:
            referrer = users.find_one({'user_id': referrer_id})
            if referrer:
                referred = users.find_one({'user_id': referred_id})
                if not referred:
                    users.insert_one({'user_id': referred_id, 'radiance': 0})
                referrals.insert_one({'referrer_id': referrer_id, 'referred_id': referred_id})
                users.update_one({'user_id': referrer_id}, {'$inc': {'radiance': 2}})
                context.bot.send_message(chat_id=referrer_id, text=f"User {update.message.from_user.username} has joined, and you earned 2 Radiance points!")

# Display user's balance
def balance(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    user = users.find_one({'user_id': user_id})
    radiance = user['radiance'] if user else 0
    update.message.reply_text(f"Your total balance is {radiance} Radiance points.")

# Handle withdrawal requests
def withdraw(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    user = users.find_one({'user_id': user_id})
    if user:
        radiance = user['radiance']
        if radiance >= 50:
            keyboard = [
                [InlineKeyboardButton("3 months (1200 Radiance)", callback_data='withdraw_3')],
                [InlineKeyboardButton("6 months (2400 Radiance)", callback_data='withdraw_6')],
                [InlineKeyboardButton("1 year (4800 Radiance)", callback_data='withdraw_12')]
            ]
            reply_markup = InlineKeyboardMarkup(keyboard)
            update.message.reply_text('Select a premium plan to withdraw:', reply_markup=reply_markup)
        else:
            update.message.reply_text("Insufficient balance to withdraw.")
    else:
        update.message.reply_text("You have no balance to withdraw.")

def withdraw_callback(update: Update, context: CallbackContext) -> None:
    query = update.callback_query
    user_id = query.from_user.id
    user = users.find_one({'user_id': user_id})
    radiance = user['radiance']

    if query.data == 'withdraw_3' and radiance >= 1200:
        users.update_one({'user_id': user_id}, {'$inc': {'radiance': -1200}})
        withdrawals.insert_one({'user_id': user_id, 'plan': '3 months', 'status': 'Pending'})
        context.bot.send_message(chat_id=user_id, text="Withdrawal request for 3 months premium plan submitted.")

    elif query.data == 'withdraw_6' and radiance >= 2400:
        users.update_one({'user_id': user_id}, {'$inc': {'radiance': -2400}})
        withdrawals.insert_one({'user_id': user_id, 'plan': '6 months', 'status': 'Pending'})
        context.bot.send_message(chat_id=user_id, text="Withdrawal request for 6 months premium plan submitted.")

    elif query.data == 'withdraw_12' and radiance >= 4800:
        users.update_one({'user_id': user_id}, {'$inc': {'radiance': -4800}})
        withdrawals.insert_one({'user_id': user_id, 'plan': '1 year', 'status': 'Pending'})
        context.bot.send_message(chat_id=user_id, text="Withdrawal request for 1 year premium plan submitted.")

    else:
        context.bot.send_message(chat_id=user_id, text="Insufficient balance or invalid selection.")

# Daily bonus handler
def daily_bonus(update: Update, context: CallbackContext) -> None:
    user_id = update.message.from_user.id
    user = users.find_one({'user_id': user_id})
    if user:
        now = datetime.now()
        last_claimed = user.get('last_claimed')
        if not last_claimed or now - last_claimed >= timedelta(days=1):
            users.update_one({'user_id': user_id}, {'$inc': {'radiance': 1}, '$set': {'last_claimed': now}})
            update.message.reply_text("You have claimed your daily bonus of 1 Radiance point.")
        else:
            update.message.reply_text("You can only claim your daily bonus once every 24 hours.")
    else:
        users.insert_one({'user_id': user_id, 'radiance': 1, 'last_claimed': datetime.now()})
        update.message.reply_text("You have claimed your daily bonus of 1 Radiance point.")

# Leaderboard handler
def leaderboard(update: Update, context: CallbackContext) -> None:
    top_users = users.find().sort('radiance', -1).limit(10)
    leaderboard_text = "Top 10 users:\n\n"
    for rank, user in enumerate(top_users, start=1):
        leaderboard_text += f"{rank}. @{user['username']} - {user['radiance']} Radiance\n"
    update.message.reply_text(leaderboard_text)

# Custom event handler
def create_event(update: Update, context: CallbackContext) -> None:
    if is_owner(update.message.from_user.id):
        event_details = " ".
