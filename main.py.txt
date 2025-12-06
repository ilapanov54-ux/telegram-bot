import asyncio
from aiogram import Bot, Dispatcher, types
from aiogram.filters import CommandStart, Command

import os

TOKEN = os.getenv("BOT_TOKEN")  # токен берём из Render переменных окружения

bot = Bot(token=TOKEN)
dp = Dispatcher()

# Пользователь → Оператор
waiting_for_registration = {}   # user_id → status


### ====== 1. Пользователь пишет START ======
@dp.message(CommandStart())
async def start_cmd(msg: types.Message):
    await msg.answer(
        "Добро пожаловать!\n\n"
        "Для регистрации отправьте:\n"
        "1) Свой ник Minecraft\n"
        "2) Свой Telegram ID (автоматически: " + str(msg.from_user.id) + ")\n\n"
        "Ожидайте, пока оператор подтвердит регистрацию."
    )

    waiting_for_registration[msg.from_user.id] = "waiting"


### ====== 2. Сообщение оператора пользователю ======
OPERATOR_ID = 999999999   # <-- СЮДА ВСТАВЬ СВОЙ TG ID!!!

@dp.message(Command(commands=["send"]))
async def operator_send(msg: types.Message):
    if msg.from_user.id != OPERATOR_ID:
        return

    try:
        _, user_id, *text = msg.text.split(" ")
        user_id = int(user_id)
        text = " ".join(text)
    except:
        await msg.answer("Формат: /send <user_id> <текст>")
        return

    await bot.send_message(user_id, f"📩 Сообщение от оператора:\n{text}")
    await msg.answer("Отправлено.")


### ====== 3. Пересылка сообщений от пользователя оператору ======
@dp.message()
async def user_message(msg: types.Message):
    if msg.from_user.id == OPERATOR_ID:
        return

    if waiting_for_registration.get(msg.from_user.id) == "waiting":
        await bot.send_message(
            OPERATOR_ID,
            f"📥 Новый запрос на регистрацию!\n"
            f"Имя: @{msg.from_user.username}\n"
            f"UserID: {msg.from_user.id}\n\n"
            f"Сообщение:\n{msg.text}"
        )
        await msg.answer("Ваши данные отправлены оператору.")
        waiting_for_registration[msg.from_user.id] = "sent"
    else:
        await bot.send_message(
            OPERATOR_ID,
            f"📨 Сообщение от {msg.from_user.id}:\n{msg.text}"
        )


async def main():
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())
