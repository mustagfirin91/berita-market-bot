import logging
import datetime
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

# === KONFIGURASI ===
TOKEN = "7832958861:AAE3ARVBW5nd7CdTWHQewHXBFoOzRW1Uhn0"
BERITA_LIST = []
ADMIN_ID = None  # Akan otomatis di-set saat user pertama kali pakai

# === LOGGING ===
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# === COMMAND ===
async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    global ADMIN_ID
    user_id = update.effective_user.id
    if ADMIN_ID is None:
        ADMIN_ID = user_id
    await update.message.reply_text(
        "✅ Bot berita market siap!\n"
        "Gunakan perintah:\n"
        "/addberita HH:MM isi_berita → tambah berita\n"
        "/listberita → lihat semua berita\n"
        "/hapusberita N → hapus berita ke-N"
    )

async def add_berita(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return await update.message.reply_text("🚫 Hanya admin yang bisa menambahkan berita.")
    try:
        jam = context.args[0]
        isi = ' '.join(context.args[1:])
        waktu = datetime.datetime.strptime(jam, "%H:%M").time()
        BERITA_LIST.append({"jam": waktu, "teks": isi})
        await update.message.reply_text(f"📝 Berita ditambahkan: {jam} - {isi}")
    except Exception as e:
        await update.message.reply_text("❌ Format salah. Contoh: /addberita 08:00 Dolar menguat karena CPI.")

async def list_berita(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if not BERITA_LIST:
        return await update.message.reply_text("📭 Belum ada berita yang ditambahkan.")
    teks = "🗂️ Daftar Berita:\n"
    for i, b in enumerate(BERITA_LIST, 1):
        teks += f"{i}. {b['jam'].strftime('%H:%M')} - {b['teks']}\n"
    await update.message.reply_text(teks)

async def hapus_berita(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if update.effective_user.id != ADMIN_ID:
        return await update.message.reply_text("🚫 Hanya admin yang bisa menghapus berita.")
    try:
        idx = int(context.args[0]) - 1
        if 0 <= idx < len(BERITA_LIST):
            b = BERITA_LIST.pop(idx)
            await update.message.reply_text(f"🗑️ Berita dihapus: {b['jam'].strftime('%H:%M')} - {b['teks']}")
        else:
            await update.message.reply_text("❌ Nomor berita tidak ditemukan.")
    except:
        await update.message.reply_text("❌ Format salah. Contoh: /hapusberita 1")

# === CEK BERITA OTOMATIS ===
async def cek_berita(context: ContextTypes.DEFAULT_TYPE):
    now = datetime.datetime.now().time().replace(second=0, microsecond=0)
    kirim = [b for b in BERITA_LIST if b['jam'] == now]
    for b in kirim:
        try:
            await context.bot.send_message(
                chat_id=ADMIN_ID,
                text=f"📅 {b['jam'].strftime('%H:%M')} WIB\n\n📌 {b['teks']}\n\n— by @FxNewsProBot"
            )
        except Exception as e:
            logger.error("Gagal kirim berita: %s", e)

# === SETUP BOT ===
app = ApplicationBuilder().token(TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(CommandHandler("addberita", add_berita))
app.add_handler(CommandHandler("listberita", list_berita))
app.add_handler(CommandHandler("hapusberita", hapus_berita))
app.job_queue.run_repeating(cek_berita, interval=60, first=5)

if __name__ == '__main__':
    print("🚀 Bot siap dijalankan!")
    app.run_polling()
