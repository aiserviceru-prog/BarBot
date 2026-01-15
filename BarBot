import os
import re
import sqlite3
from datetime import datetime
from fastapi import FastAPI, Request
from aiogram import Bot, Dispatcher
from aiogram.types import Update, Message, CallbackQuery, KeyboardButton, ReplyKeyboardMarkup, InlineKeyboardMarkup, InlineKeyboardButton
from aiogram.fsm.state import State, StatesGroup
from aiogram.fsm.context import FSMContext
import asyncio

# ---------------- Настройки ----------------
TOKEN = os.getenv("8413148431:AAHhjWJuxY29QuaA_pUE7b0ue2tWgnQtboA")  # Telegram Bot Token
WEBHOOK_URL = os.getenv("WEBHOOK_URL")  # https://project.up.railway.app
bot = Bot(TOKEN)
dp = Dispatcher(bot)
app = FastAPI()

# ---------------- FSM ----------------------
class EditOrder(StatesGroup):
    waiting_for_edit = State()

# ---------------- SQLite -------------------
conn = sqlite3.connect("orders.db", check_same_thread=False)
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS orders (
    item TEXT PRIMARY KEY,
    quantity REAL
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT,
    user_id INTEGER,
    username TEXT,
    action TEXT,
    details TEXT
)
""")
conn.commit()

order_storage = {}

def load_orders():
    cursor.execute("SELECT item, quantity FROM orders")
    for item, qty in cursor.fetchall():
        order_storage[item] = qty

def save_orders():
    cursor.execute("DELETE FROM orders")
    for item, qty in order_storage.items():
        cursor.execute("INSERT INTO orders (item, quantity) VALUES (?, ?)", (item, qty))
    conn.commit()

def log_action(user, action: str, details: str = ""):
    cursor.execute(
        "INSERT INTO logs (timestamp, user_id, username, action, details) VALUES (?, ?, ?, ?, ?)",
        (datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
         user.id, user.username or user.full_name, action, details)
    )
    conn.commit()

# ---------------- Клавиатуры ----------------
BTN_SHOW = "📋 Показать общий заказ"
BTN_EDIT = "✏️ Редактировать заказ"
BTN_CLEAR = "🗑 Очистить заказ"

reply_kb = ReplyKeyboardMarkup(
    keyboard=[
        [KeyboardButton(BTN_SHOW)],
        [KeyboardButton(BTN_EDIT)],
        [KeyboardButton(BTN_CLEAR)]
    ],
    resize_keyboard=True
)

clear_kb = InlineKeyboardMarkup(
    inline_keyboard=[
        [
            InlineKeyboardButton("✅ Да", callback_data="clear_yes"),
            InlineKeyboardButton("❌ Нет", callback_data="clear_no")
        ]
    ]
)

# ---------------- Парсер -------------------
def parse_order(text: str) -> dict[str, float]:
    items = {}
    for line in text.splitlines():
        line = line.strip()
        if not line:
            continue
        m = re.search(r"(.+?)\s+([\d.,]+)", line)
        if not m:
            continue
        name = m.group(1).strip().lower()
        qty = float(m.group(2).replace(",", "."))
        items[name] = items.get(name, 0) + qty
    return items

def format_order(raw=False):
    if not order_storage:
        return ""
    lines = [f"{k.capitalize()} {v}" for k, v in order_storage.items()]
    return "\n".join(lines) if raw else "📦 Общий заказ:\n\n" + "\n".join(f"• {l}" for l in lines)

# ---------------- Handlers -------------------
@dp.message()
async def all_messages(message: Message, state: FSMContext):
    # Если редактирование
    if await state.get_state() == EditOrder.waiting_for_edit:
        parsed = parse_order(message.text)
        if not parsed:
            await message.answer("⚠️ Не удалось распознать заказ.")
            return
        order_storage.clear()
        order_storage.update(parsed)
        save_orders()
        log_action(message.from_user, "EDIT_ORDER", f"Новый заказ: {message.text}")
        await state.clear()
        await message.answer("✅ Заказ полностью перезаписан.")
        return

    # Кнопки
    if message.text == BTN_SHOW:
        await message.answer(format_order() or "❌ Заказ пуст.")
        log_action(message.from_user, "SHOW_ORDER")
        return
    if message.text == BTN_EDIT:
        if not order_storage:
            await message.answer("❌ Заказ пуст.")
            return
        await state.set_state(EditOrder.waiting_for_edit)
        await message.answer(
            "✏️ Ответь на это сообщение, скопируй и отредактируй заказ. Старый заказ будет полностью заменён."
        )
        await message.answer(format_order(raw=True))
        return
    if message.text == BTN_CLEAR:
        if not order_storage:
            await message.answer("ℹ️ Заказ уже пуст.")
            return
        await message.answer("⚠️ Очистить весь заказ?", reply_markup=clear_kb)
        return

    # Добавление новых позиций
    parsed = parse_order(message.text)
    if parsed:
        for k, v in parsed.items():
            order_storage[k] = order_storage.get(k, 0) + v
        save_orders()
        log_action(message.from_user, "ADD_ITEMS", message.text.replace("\n","; "))
        await message.answer("✅ Добавлено в общий заказ.")

# ---------------- Callback для очистки ----------------
@dp.callback_query()
async def callbacks(call: CallbackQuery):
    if call.data == "clear_yes":
        order_storage.clear()
        save_orders()
        log_action(call.from_user, "CLEAR_ORDER")
        await call.message.edit_text("🗑 Заказ очищен.")
        await call.answer()
    elif call.data == "clear_no":
        await call.message.edit_text("❌ Очистка отменена.")
        await call.answer()

# ---------------- Webhook endpoint ----------------
@app.post(f"/webhook/{TOKEN}")
async def telegram_webhook(req: Request):
    data = await req.json()
    update = Update(**data)
    await dp.process_update(update)
    return {"ok": True}

# ---------------- Startup / Shutdown ----------------
@app.on_event("startup")
async def on_startup():
    load_orders()
    await bot.set_webhook(f"{WEBHOOK_URL}/webhook/{TOKEN}")

@app.on_event("shutdown")
async def on_shutdown():
    await bot.session.close()

# ---------------- Запуск ----------------
if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=int(os.environ.get("PORT", 8000)))
