import telebot
from telebot import types
import os
from flask import Flask
import threading
import time

# --- НАСТРОЙКИ ---
BOT_TOKEN = os.environ.get('BOT_TOKEN') 
bot = telebot.TeleBot(BOT_TOKEN)

# Хранилище данных пользователей
user_data = {}

# --- ЗАЩИТА ОТ СПАМА (АНТИФЛУД) ---
user_last_message_time = {}
ANTI_FLOOD_DELAY = 1.0  # Задержка в секундах между сообщениями

class AntiFloodMiddleware:
    def __init__(self, bot):
        self.bot = bot

    def __call__(self, message):
        user_id = message.from_user.id
        current_time = time.time()
        
        if user_id in user_last_message_time:
            last_time = user_last_message_time[user_id]
            if current_time - last_time < ANTI_FLOOD_DELAY:
                return None  # Игнорируем спам
                
        user_last_message_time[user_id] = current_time
        return message

bot.setup_middleware(AntiFloodMiddleware(bot))

# --- КОМАНДА СТАРТ ---
@bot.message_handler(commands=['start'])
def start(message):
    markup = types.ReplyKeyboardMarkup(resize_keyboard=True)
    btn_start_calc = types.KeyboardButton('🚀 Начать новый расчет')
    markup.add(btn_start_calc)
    
    bot.send_message(
        message.chat.id, 
        "👋 Привет! Я бот-калькулятор для расчета заказов.\n\nНажми кнопку ниже, чтобы начать.", 
        reply_markup=markup
    )

@bot.message_handler(func=lambda message: message.text == '🚀 Начать новый расчет')
def start_calc(message):
    user_id = message.from_user.id
    user_data[user_id] = {}  # Очищаем данные для нового расчета
    
    markup = types.ReplyKeyboardRemove()
    msg = bot.send_message(
        message.chat.id, 
        "Введи **Имя заказчика и номер заказа** (например: *Иван, Заказ №123*):", 
        parse_mode='Markdown',
        reply_markup=markup
    )
    bot.register_next_step_handler(msg, process_customer_info)

# --- СБОР ДАННЫХ ---
def process_customer_info(message):
    user_id = message.from_user.id
    user_data[user_id]['customer_info'] = message.text
    
    msg = bot.send_message(message.chat.id, "🛒 С какой площадки заказ? (например: *Pinduoduo, Taobao, 1688*)", parse_mode='Markdown')
    bot.register_next_step_handler(msg, process_marketplace)

def process_marketplace(message):
    user_id = message.from_user.id
    user_data[user_id]['marketplace'] = message.text
    
    msg = bot.send_message(message.chat.id, "📦 Сколько всего **позиций** в заказе?")
    bot.register_next_step_handler(msg, process_items_count)

def process_items_count(message):
    user_id = message.from_user.id
    user_data[user_id]['items_count'] = message.text
    
    msg = bot.send_message(message.chat.id, "💰 Введи **общую стоимость товаров в юанях** (только цифры, например: *625.7*):")
    bot.register_next_step_handler(msg, process_total_cny)

def process_total_cny(message):
    user_id = message.from_user.id
    try:
        total_cny = float(message.text.replace(',', '.'))
        user_data[user_id]['total_cny'] = total_cny
        
        # Выбор сценария
        markup = types.InlineKeyboardMarkup(row_width=1)
        btn_standard = types.InlineKeyboardButton("💱 Через тенге (Юань → Тенге → Рубль)", callback_data="scenario_kzt")
        btn_direct = types.InlineKeyboardButton("💱 Напрямую (Юань → Рубль)", callback_data="scenario_rub")
        markup.add(btn_standard, btn_direct)
        
        bot.send_message(message.chat.id, "Выбери, как будем считать стоимость:", reply_markup=markup)
        
    except ValueError:
        msg = bot.send_message(message.chat.id, "❌ Ошибка! Введи число (например: 625.7). Попробуй еще раз:")
        bot.register_next_step_handler(msg, process_total_cny)

# --- ОБРАБОТКА ВЫБОРА СЦЕНАРИЯ ---
@bot.callback_query_handler(func=lambda call: call.data in ['scenario_kzt', 'scenario_rub'])
def callback_scenario(call):
    user_id = call.from_user.id
    chat_id = call.message.chat.id
    
    bot.delete_message(chat_id, call.message.message_id) # Удаляем сообщение с кнопками
    
    if call.data == 'scenario_kzt':
        user_data[user_id]['scenario'] = 'kzt'
        markup = types.ReplyKeyboardMarkup(one_time_keyboard=True, resize_keyboard=True)
        markup.add(types.KeyboardButton('Пропустить'))
        msg = bot.send_message(chat_id, "💱 Введи **курс юаня к тенге** (например: *67.65*)\nИли нажми «Пропустить»:", reply_markup=markup, parse_mode='Markdown')
        bot.register_next_step_handler(msg, process_rate_cny_kzt)
        
    elif call.data == 'scenario_rub':
        user_data[user_id]['scenario'] = 'rub'
        markup = types.ReplyKeyboardMarkup(one_time_keyboard=True, resize_keyboard=True)
        markup.add(types.KeyboardButton('Пропустить'))
        msg = bot.send_message(chat_id, "💱 Введи **курс юаня к рублю** (например: *12.5*)\nИли нажми «Пропустить»:", reply_markup=markup, parse_mode='Markdown')
        bot.register_next_step_handler(msg, process_rate_cny_rub)

# --- ВЕТКА 1: ЧЕРЕЗ ТЕНГЕ ---
def process_rate_cny_kzt(message):
    user_id = message.from_user.id
    
    if message.text == 'Пропустить':
        user_data[user_id]['rate_cny_kzt'] = None
    else:
        try:
            user_data[user_id]['rate_cny_kzt'] = float(message.text.replace(',', '.'))
        except ValueError:
            msg = bot.send_message(message.chat.id, "❌ Ошибка! Введи число или нажми «Пропустить». Попробуй еще раз:")
            bot.register_next_step_handler(msg, process_rate_cny_kzt)
            return

    markup = types.ReplyKeyboardMarkup(one_time_keyboard=True, resize_keyboard=True)
    markup.add(types.KeyboardButton('Пропустить'))
    msg = bot.send_message(message.chat.id, "💱 Введи **курс тенге к рублю** (например: *5.08*)\nИли нажми «Пропустить»:", reply_markup=markup, parse_mode='Markdown')
    bot.register_next_step_handler(msg, process_rate_kzt_rub)

def process_rate_kzt_rub(message):
    user_id = message.from_user.id
    
    if message.text == 'Пропустить':
        user_data[user_id]['rate_kzt_rub'] = None
    else:
        try:
            user_data[user_id]['rate_kzt_rub'] = float(message.text.replace(',', '.'))
        except ValueError:
            msg = bot.send_message(message.chat.id, "❌ Ошибка! Введи число или нажми «Пропустить». Попробуй еще раз:")
            bot.register_next_step_handler(msg, process_rate_kzt_rub)
            return

    msg = bot.send_message(message.chat.id, "📈 Введи **процент наценки/запаса** (например: *10* для 10%):", reply_markup=types.ReplyKeyboardRemove())
    bot.register_next_step_handler(msg, process_markup)

# --- ВЕТКА 2: НАПРЯМУЮ В РУБЛЬ ---
def process_rate_cny_rub(message):
    user_id = message.from_user.id
    
    if message.text == 'Пропустить':
        user_data[user_id]['rate_cny_rub'] = None
    else:
        try:
            user_data[user_id]['rate_cny_rub'] = float(message.text.replace(',', '.'))
        except ValueError:
            msg = bot.send_message(message.chat.id, "❌ Ошибка! Введи число или нажми «Пропустить». Попробуй еще раз:")
            bot.register_next_step_handler(msg, process_rate_cny_rub)
            return

    msg = bot.send_message(message.chat.id, "📈 Введи **процент наценки/запаса** (например: *10* для 10%):", reply_markup=types.ReplyKeyboardRemove())
    bot.register_next_step_handler(msg, process_markup)

# --- ФИНАЛЬНЫЙ РАСЧЕТ ---
def process_markup(message):
    user_id = message.from_user.id
    try:
        markup = float(message.text.replace(',', '.'))
        user_data[user_id]['markup'] = markup
        
        data = user_data[user_id]
        scenario = data.get('scenario')
        
        total_cny_with_markup = data['total_cny'] * (1 + markup / 100)
        
        total_kzt = None
        total_rub = None
        
        if scenario == 'kzt':
            rate_cny_kzt = data.get('rate_cny_kzt')
            rate_kzt_rub = data.get('rate_kzt_rub')
            
            if rate_cny_kzt is None or rate_kzt_rub is None:
                markup_start = types.ReplyKeyboardMarkup(resize_keyboard=True)
                markup_start.add(types.KeyboardButton('🚀 Начать новый расчет'))
                bot.send_message(message.chat.id, "❌ Ошибка: для расчета через тенге нужны оба курса.", reply_markup=markup_start)
                return
                
            total_kzt = total_cny_with_markup * rate_cny_kzt
            total_rub = total_kzt / rate_kzt_rub
            calc_method = "через тенге"
            
        elif scenario == 'rub':
            rate_cny_rub = data.get('rate_cny_rub')
            
            if rate_cny_rub is None:
                markup_start = types.ReplyKeyboardMarkup(resize_keyboard=True)
                markup_start.add(types.KeyboardButton('🚀 Начать новый расчет'))
                bot.send_message(message.chat.id, "❌ Ошибка: для прямого расчета нужен курс юаня к рублю.", reply_markup=markup_start)
                return
                
            total_rub = total_cny_with_markup * rate_cny_rub
            calc_method = "напрямую"
        
        # Формирование красивого ответа
        result_text = (
            f"✅ **РАСЧЕТ ГОТОВ ({calc_method})**\n\n"
            f"👤 **Заказчик:** {data['customer_info']}\n"
            f"🛒 **Площадка:** {data['marketplace']}\n"
            f"📦 **Позиций:** {data['items_count']}\n"
            f"———————————————\n"
            f"💴 **Товары (юани):** {data['total_cny']:.2f} ¥\n"
            f"📈 **С наценкой {markup}%:** {total_cny_with_markup:.2f} ¥\n"
            f"———————————————\n"
        )
        
        if scenario == 'kzt':
            result_text += f"💱 **Курс ¥/₸:** {data['rate_cny_kzt']:.2f}\n"
            result_text += f"💱 **Курс ₸/₽:** {data['rate_kzt_rub']:.2f}\n"
        else:
            result_text += f"💱 **Курс ¥/₽:** {data['rate_cny_rub']:.2f}\n"
            
        result_text += "———————————————\n"
        
        if total_kzt is not None:
            result_text += f"🇰🇿 **К оплате в тенге:** {total_kzt:.2f} ₸\n"
        if total_rub is not None:
            result_text += f"🇷🇺 **Запросить с клиента:** {total_rub:.2f} ₽"
        
        markup_start = types.ReplyKeyboardMarkup(resize_keyboard=True)
        markup_start.add(types.KeyboardButton('🚀 Начать новый расчет'))
        
        bot.send_message(message.chat.id, result_text, parse_mode='Markdown', reply_markup=markup_start)
        
    except ValueError:
        msg = bot.send_message(message.chat.id, "❌ Ошибка! Введи число (например: 10). Попробуй еще раз:")
        bot.register_next_step_handler(msg, process_markup)


# --- МИНИ-ВЕБ-СЕРВЕР ДЛЯ RENDER ---
app = Flask(__name__)

@app.route('/')
def index():
    return "Bot is running!"

def run_flask():
    port = int(os.environ.get('PORT', 10000))
    app.run(host='0.0.0.0', port=port)

# --- ЗАПУСК ---
if __name__ == '__main__':
    threading.Thread(target=run_flask).start()
    print("Бот запущен и работает...")
    bot.infinity_polling()
