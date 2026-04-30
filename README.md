import telebot
from telebot.types import InlineKeyboardMarkup, InlineKeyboardButton
import yt_dlp
import os
import subprocess

TOKEN = '8770442673:AAGpTPcUJnqXWocfXMGsWAt3sTvgIyG2K68'
bot = telebot.TeleBot(TOKEN)

user_data = {}

class UserState:
    def __init__(self, link):
        self.link = link
        self.type = None  
        self.is_cut = None  
        self.cut_time = None
        self.file_name = None

@bot.message_handler(commands=['start'])
def send_welcome(message):
    bot.reply_to(message, "أهلاً يا عمار! ابعت اللينك وهظبطك 🫡")

@bot.message_handler(func=lambda message: message.text.startswith('http'))
def process_link(message):
    chat_id = message.chat.id
    user_data[chat_id] = UserState(message.text)
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(InlineKeyboardButton("فيديو 🎥", callback_data="type_video"), 
               InlineKeyboardButton("صوت 🎵", callback_data="type_audio"))
    bot.reply_to(message, "نوع الملف إيه؟", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data in ['type_video', 'type_audio'])
def process_type(call):
    chat_id = call.message.chat.id
    user_data[chat_id].type = 'video' if call.data == 'type_video' else 'audio'
    markup = InlineKeyboardMarkup(row_width=2)
    markup.add(InlineKeyboardButton("كامل 🟢", callback_data="cut_full"), 
               InlineKeyboardButton("يتقص ✂️", callback_data="cut_yes"))
    bot.edit_message_text(chat_id=chat_id, message_id=call.message.message_id, text="كامل ولا قص؟", reply_markup=markup)

@bot.callback_query_handler(func=lambda call: call.data in ['cut_full', 'cut_yes'])
def process_cut(call):
    chat_id = call.message.chat.id
    user_data[chat_id].is_cut = 'full' if call.data == 'cut_full' else 'cut'
    if user_data[chat_id].is_cut == 'cut':
        msg = bot.send_message(chat_id, "اكتب الوقت (بداية-نهاية)\nمثلاً: 00:05-00:15")
        bot.register_next_step_handler(msg, process_cut_time)
    else:
        msg = bot.send_message(chat_id, "اسم الملف إيه؟")
        bot.register_next_step_handler(msg, process_name)

def process_cut_time(message):
    user_data[message.chat.id].cut_time = message.text
    msg = bot.send_message(message.chat.id, "اسم الملف إيه؟")
    bot.register_next_step_handler(msg, process_name)

def process_name(message):
    chat_id = message.chat.id
    user_data[chat_id].file_name = message.text
    data = user_data[chat_id]
    bot.send_message(chat_id, "جاري التنفيذ.. انتظر قليلاً ⏳")
    try:
        ydl_opts = {'quiet': True, 'nocheckcertificate': True, 'outtmpl': 'raw_file.%(ext)s'}
        if data.type == 'audio':
            ydl_opts['format'] = 'bestaudio/best'
            ydl_opts['postprocessors'] = [{'key': 'FFmpegExtractAudio', 'preferredcodec': 'mp3', 'preferredquality': '192'}]
            actual_raw = 'raw_file.mp3'
            final_file = f"{data.file_name}.mp3"
        else:
            ydl_opts['format'] = 'best[ext=mp4]/best'
            actual_raw = 'raw_file.mp4'
            final_file = f"{data.file_name}.mp4"

        with yt_dlp.YoutubeDL(ydl_opts) as ydl:
            ydl.extract_info(data.link, download=True)

        if data.is_cut == 'cut' and '-' in data.cut_time:
            start, end = data.cut_time.split('-')
            cmd = ['ffmpeg', '-y', '-i', actual_raw, '-ss', start.strip(), '-to', end.strip(), '-c', 'copy', final_file]
            subprocess.run(cmd, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
            if os.path.exists(actual_raw): os.remove(actual_raw)
        else:
            os.rename(actual_raw, final_file)

        with open(final_file, 'rb') as f:
            if data.type == 'audio': bot.send_audio(chat_id, f)
            else: bot.send_video(chat_id, f)
        os.remove(final_file)
        bot.send_message(chat_id, "تم بنجاح! ✅")
    except Exception as e:
        bot.send_message(chat_id, f"خطأ: {str(e)}")

bot.infinity_polling(skip_pending=True)
