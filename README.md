scan_progress.json fayli orqali progressni saqlash, qayta ishga tushganda o'tkazib yuborish va FloodWaitError istisnosini to'g'ri ushlash kodi quyidagicha amalga oshiriladi:

Python
import json
import os
import sys
import asyncio
from dotenv import load_dotenv
from telethon import TelegramClient
from telethon.errors import FloodWaitError

load_dotenv()

API_ID = os.getenv("API_ID")
API_HASH = os.getenv("API_HASH")

PROGRESS_FILE = "scan_progress.json"

if not API_ID or not API_HASH:
    print("Xatolik: API_ID yoki API_HASH .env faylida topilmadi!")
    sys.exit(1)

# Progressni fayldan o'qish funksiyasi
def load_progress():
    if os.path.exists(PROGRESS_FILE):
        try:
            with open(PROGRESS_FILE, "r", encoding="utf-8") as f:
                return json.load(f)
        except json.JSONDecodeError:
            pass
    return {"last_scanned_id": 0, "scanned_ids": []}

# Progressni faylga saqlash funksiyasi
def save_progress(data):
    with open(PROGRESS_FILE, "w", encoding="utf-8") as f:
        json.dump(data, f, indent=4)

client = TelegramClient("scanner_session", int(API_ID), API_HASH)

async def scan_messages():
    progress = load_progress()
    last_scanned_id = progress.get("last_scanned_id", 0)
    scanned_ids = set(progress.get("scanned_ids", []))

    target_chat = "me"  # Skanlanadigan chat yoki kanal
    print(f"Skanlash boshlandi. Oxirgi ishlov berilgan xabar ID: {last_scanned_id}")

    try:
        await client.connect()
        if not await client.is_user_authorized():
            print("Xatolik: Avtorizatsiyadan o'tilmagan!")
            return

        # min_id parametri orqali faqat oxirgi saqlangan ID dan keyingi xabarlar olinadi
        async for message in client.iter_messages(target_chat, min_id=last_scanned_id, reverse=True):
            
            # Takroriy xabarlarni qayta ishlamaslik tekshiruvi
            if message.id in scanned_ids:
                continue

            try:
                # Xabar ustida bajariladigan amallar (masalan, chop etish)
                print(f"[Yangi xabar] ID: {message.id} | Matn: {message.text}")

                # Progress ma'lumotlarini yangilash
                scanned_ids.add(message.id)
                progress["last_scanned_id"] = message.id
                progress["scanned_ids"] = list(scanned_ids)
                
                # Faylga saqlash
                save_progress(progress)
                
                await asyncio.sleep(0.5)

            except FloodWaitError as e:
                # Alohida xabar ishlanayotganda FloodWait yuzaga kelsa
                print(f"[FloodWaitError] Telegram limit belgiladi. {e.seconds} soniya kutilmoqda...")
                await asyncio.sleep(e.seconds)

    except FloodWaitError as e:
        # iter_messages jarayonida FloodWait yuzaga kelsa
        print(f"[FloodWaitError] Umumiy so'rov bo'yicha limit: {e.seconds} soniya kutilmoqda...")
        await asyncio.sleep(e.seconds)
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(scan_messages())
Asosiy imkoniyatlar:

Progress holati: scan_progress.json faylida oxirgi skan qilingan xabar ID si (last_scanned_id) va ishlov berilgan barcha ID lar ro'yxati saqlanadi.

Qayta ishlamaslik (Idempotentlik): iter_messages(..., min_id=last_scanned_id) hamda if message.id in scanned_ids: tekshiruvi dastur qayta ishga tushganda eski xabarlar takroran qayta ishlanishining oldini oladi.

FloodWaitError ushlash: Telegram tomonidan so'rovlar limiti oshib ketganda FloodWaitError tutib qolinadi va e.seconds daqiqalari bo'yicha avtomatik tanaffus qilinadi.
