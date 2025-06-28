import logging
from telegram import ReplyKeyboardMarkup, KeyboardButton, ReplyKeyboardRemove, Update, InputFile, InlineKeyboardButton, InlineKeyboardMarkup
import os
import uuid
import json
import io
import qrcode
from telegram import InputFile
import random
from fake_useragent import UserAgent
from faker import Faker
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, CallbackQueryHandler, ConversationHandler
from datetime import datetime
import pytz
from dotenv import load_dotenv
import asyncio
import httpx
from flask import Flask
import threading

load_dotenv()

app = Flask(__name__)

@app.route('/')
def home():
    return "✅ Bot Telegram Sedang Berjalan!", 200

def run_flask():
    """Menjalankan server Flask di thread terpisah."""
    app.run(host="0.0.0.0", port=8080)

def start_keep_alive():
    """Menjalankan keep_alive() di thread terpisah."""
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    loop.run_until_complete(keep_alive())

async def keep_alive():
    """Mengirim ping ke API Telegram setiap 5 menit."""
    while True:
        try:
            url = f"https://api.telegram.org/bot{BOT_TOKEN}/getMe"
            async with httpx.AsyncClient() as client:
                response = await client.get(url, timeout=10)
        except Exception as e:
            print(f"❌ Ping error: {e}", flush=True)
        await asyncio.sleep(50)

TESTIMONI_URL = os.getenv("TESTIMONI_URL")
ADMIN_URL = os.getenv("ADMIN_URL")
STORE_NAME = os.getenv("STORE_NAME")
CHANNEL_URL = os.getenv("CHANNEL_URL")
# Inisialisasi ADMIN_BOT di luar fungsi main
ADMIN_NOTIFICATION_BOT_TOKEN = None
ADMIN_BOT = None


try:
    with open(os.path.join("TELEGRAM", "config.json"), "r") as config_file:
        config = json.load(config_file)
    BOT_TOKEN = config.get("bot_token")
    ADMIN_NOTIFICATION_BOT_TOKEN = config.get("admin_notification_bot_token") # Tambahkan baris ini

    if not BOT_TOKEN:
        raise ValueError("Pastikan bot_token diisi dalam config.json")

    # Inisialisasi ADMIN_BOT jika tokennya ada
    if ADMIN_NOTIFICATION_BOT_TOKEN:
        from telegram import Bot
        ADMIN_BOT = Bot(token=ADMIN_NOTIFICATION_BOT_TOKEN)
        print("✅ Admin Notification Bot diinisialisasi.")
    else:
        print("⚠️ admin_notification_bot_token tidak ditemukan di config.json. Notifikasi admin via bot terpisah tidak akan berfungsi.")

except (FileNotFoundError, json.JSONDecodeError, ValueError) as e:
    print(f"❌ ERROR: {e}")
    exit(1)

# File paths
PENGUNJUNG_FILE = os.path.join("VISITOR", "totalpengunjung.json")
TERJUAL_FILE = os.path.join("VISITOR", "totalterjual.json")
STOCK_FILE = "initial_stock.json"
USERS_FILE = "users.json"
QRIS_FILE = "qris.json"
TRX_FILE = "trx.json"
ORKUT_FILE = "Orkut.json"
CHAT_ID_FILE = "chatid.json"

def load_statistic(file_path):
    """Memuat data statistik dari file JSON dengan penanganan error."""
    try:
        if os.path.exists(file_path):
            with open(file_path, "r", encoding="utf-8") as file:
                data = json.load(file)
                if isinstance(data, dict) and "count" in data:
                    logging.info(f"Loaded statistic from {file_path}: {data}")
                    return data
                else:
                    logging.warning(f"Invalid format in {file_path}, initializing with count 0")
                    return {"count": 0}
        else:
            logging.info(f"File {file_path} not found, initializing with count 0")
            return {"count": 0}
    except json.JSONDecodeError as e:
        logging.error(f"Error decoding {file_path}: {e}, initializing with count 0")
        return {"count": 0}
    except Exception as e:
        logging.error(f"Unexpected error loading {file_path}: {e}, initializing with count 0")
        return {"count": 0}

def save_statistic(file_path, data):
    """Menyimpan data statistik ke file JSON."""
    try:
        os.makedirs(os.path.dirname(file_path), exist_ok=True)
        with open(file_path, "w", encoding="utf-8") as file:
            json.dump(data, file, indent=4)
        logging.info(f"Saved statistic to {file_path}: {data}")
    except Exception as e:
        logging.error(f"Error saving {file_path}: {e}")

# Muat data statistik
pengunjung_data = load_statistic(PENGUNJUNG_FILE)
terjual_data = load_statistic(TERJUAL_FILE)

def update_pengunjung():
    """Increment total pengunjung."""
    pengunjung_data["count"] += 1
    save_statistic(PENGUNJUNG_FILE, pengunjung_data)
    logging.info(f"Updated pengunjung count: {pengunjung_data['count']}")

def update_terjual(jumlah: int):
    """Menambah jumlah total terjual setelah pesanan sukses."""
    if jumlah > 0:
        terjual_data["count"] = terjual_data.get("count", 0) + jumlah
        save_statistic(TERJUAL_FILE, terjual_data)
        logging.info(f"✅ Total terjual bertambah: {jumlah}. Total sekarang: {terjual_data['count']}")

def load_initial_stock():
    """Memuat stok awal dari file JSON."""
    try:
        if os.path.exists(STOCK_FILE):
            with open(STOCK_FILE, "r", encoding="utf-8") as file:
                data = json.load(file)
                logging.info(f"Loaded initial stock: {data}")
                return data
        logging.info("initial_stock.json not found, initializing empty")
        return {}
    except json.JSONDecodeError as e:
        logging.error(f"Error decoding initial_stock.json: {e}, initializing empty")
        return {}
    except Exception as e:
        logging.error(f"Unexpected error loading initial_stock.json: {e}, initializing empty")
        return {}

def save_initial_stock(stock_data):
    """Menyimpan stok awal ke file JSON."""
    try:
        with open(STOCK_FILE, "w", encoding="utf-8") as file:
            json.dump(stock_data, file, indent=4)
        logging.info(f"Saved initial stock: {stock_data}")
    except Exception as e:
        logging.error(f"Error saving initial_stock.json: {e}")

def set_initial_stock():
    """Set stok awal saat program dijalankan."""
    stock_data = load_initial_stock()
    products = load_products()
    updated = False
    for product, data in products.items():
        if product not in stock_data:
            stock_data[product] = data["stock"]
            updated = True
            logging.info(f"Initialized stock for {product}: {data['stock']}")
    if updated:
        save_initial_stock(stock_data)

def load_products():
    """Memuat daftar produk, harga, stok, dan deskripsi."""
    products = {}
    try:
        with open(os.path.join("PRODUCTS", "Ready.txt"), "r") as file:
            for line in file:
                parts = line.strip().split()
                if len(parts) == 2:
                    name, amount = parts
                    filename = f"{name}.txt"
                    stock = get_stock_from_file(filename)
                    desk_data = load_desk()
                    description = desk_data.get(name, "Deskripsi tidak tersedia.")
                    products[name] = {
                        "price": int(amount),
                        "filename": filename,
                        "stock": stock,
                        "desk": description
                    }
        logging.info(f"Loaded products: {list(products.keys())}")
    except FileNotFoundError:
        logging.warning("Ready.txt tidak ditemukan!")
    return products

# ... (kode yang sudah ada) ...

# Konversasi untuk ambil stok
TAKE_STOCK_PRODUCT, TAKE_STOCK_QUANTITY = range(2)

async def take_stock_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Memulai proses pengambilan stok oleh admin."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return ConversationHandler.END

    products = load_products()
    if not products:
        await update.message.reply_text("⚠️ Saat ini tidak ada produk yang tersedia untuk diambil.")
        return ConversationHandler.END

    product_list_text = "📦 *Pilih produk yang ingin diambil stoknya:*\n\n"
    keyboard = []
    for product_name, data in products.items():
        stock_info = f"({data['stock']} tersedia)" if data['stock'] > 0 else "(Kosong)"
        product_list_text += f"- {product_name} {stock_info}\n"
        keyboard.append([InlineKeyboardButton(product_name, callback_data=f"takestock_{product_name}")])

    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text(product_list_text, parse_mode="Markdown", reply_markup=reply_markup)
    return TAKE_STOCK_PRODUCT

async def take_stock_select_product(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menangani pemilihan produk untuk pengambilan stok."""
    query = update.callback_query
    await query.answer()
    product_name = query.data.split("_")[1]

    products = load_products()
    if product_name not in products:
        await query.edit_message_text("⚠️ Produk tidak ditemukan.")
        return ConversationHandler.END

    context.user_data['take_stock_product'] = product_name
    stock_available = products[product_name]['stock']

    await query.edit_message_text(
        f"✅ Anda memilih produk: *{product_name}*\n"
        f"📦 Stok tersedia: {stock_available}\n\n"
        f"🔢 Sekarang, masukkan jumlah akun yang ingin diambil (maksimal {stock_available}).\n"
        f"Gunakan `/cancel` untuk membatalkan.",
        parse_mode="Markdown"
    )
    return TAKE_STOCK_QUANTITY

async def take_stock_enter_quantity(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menangani jumlah akun yang ingin diambil."""
    if not update.message or not update.message.text:
        await update.message.reply_text("⚠️ Mohon masukkan jumlah dalam angka.")
        return TAKE_STOCK_QUANTITY

    try:
        quantity = int(update.message.text.strip())
        if quantity <= 0:
            await update.message.reply_text("⚠️ Jumlah harus lebih besar dari 0.")
            return TAKE_STOCK_QUANTITY
    except ValueError:
        await update.message.reply_text("⚠️ Jumlah harus berupa angka.")
        return TAKE_STOCK_QUANTITY

    product_name = context.user_data.get('take_stock_product')
    if not product_name:
        await update.message.reply_text("❌ Terjadi kesalahan. Produk tidak terdefinisi. Silakan ulangi dari awal dengan /ambilstok.")
        return ConversationHandler.END

    products = load_products()
    stok_tersedia = products[product_name]['stock']

    if quantity > stok_tersedia:
        await update.message.reply_text(f"⚠️ Stok {product_name} tidak mencukupi. Tersedia: {stok_tersedia}. Mohon masukkan jumlah yang valid.")
        return TAKE_STOCK_QUANTITY

    try:
        produk_file = f"{product_name}.txt"
        with open(produk_file, "r", encoding="utf-8") as file:
            akun_lines = [line.strip() for line in file if line.strip()]

        akun_terambil = akun_lines[:quantity]
        sisa_akun = akun_lines[quantity:]

        with open(produk_file, "w", encoding="utf-8") as file:
            file.write("\n".join(sisa_akun))

        # Update stock data
        stock_data = load_initial_stock()
        stock_data[product_name] = len(sisa_akun)
        save_initial_stock(stock_data)

        # Update total terjual (bisa disesuaikan jika admin ambil stok tidak dihitung sebagai 'terjual')
        update_terjual(quantity)

        akun_message = "\n".join(akun_terambil)
        filename = f"admin_take_{product_name}_{uuid.uuid4().hex[:8]}.txt"
        file_path = os.path.join("TRANSAKSI", filename) # Simpan di folder TRANSAKSI
        os.makedirs("TRANSAKSI", exist_ok=True)
        with open(file_path, "w", encoding="utf-8") as f:
            f.write(akun_message)

        await update.message.reply_document(
            document=InputFile(file_path, filename=filename),
            caption=f"✅ Admin berhasil mengambil {quantity} akun *{product_name}*.",
            parse_mode="Markdown"
        )
        await update.message.reply_text(f"Total akun *{product_name}* tersisa: {len(sisa_akun)}.")

    except Exception as e:
        await update.message.reply_text(f"❌ Terjadi kesalahan saat mengambil stok: {e}")

    # Clean up user data for this conversation
    context.user_data.pop('take_stock_product', None)
    return ConversationHandler.END

async def cancel_take_stock(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Membatalkan proses pengambilan stok."""
    context.user_data.pop('take_stock_product', None)
    await update.message.reply_text("❌ Pengambilan stok dibatalkan.", parse_mode="Markdown")
    return ConversationHandler.END

# ... (lanjutkan ke bagian main() untuk menambahkan handler) ...

def load_desk():
    """Membaca deskripsi produk dari desk.json."""
    desk_file = "desk.json"
    try:
        if os.path.exists(desk_file):
            with open(desk_file, "r", encoding="utf-8") as file:
                return json.load(file)
        return {}
    except json.JSONDecodeError as e:
        logging.error(f"Error decoding desk.json: {e}, returning empty dict")
        return {}
    except Exception as e:
        logging.error(f"Unexpected error loading desk.json: {e}, returning empty dict")
        return {}

def get_stock_from_file(filename):
    """Membaca stok produk dari file."""
    file_path = os.path.join(os.getcwd(), filename)
    try:
        with open(file_path, "r", encoding="utf-8") as file:
            stock = len([line for line in file if line.strip()])
            logging.info(f"Stock for {filename}: {stock}")
            return stock
    except FileNotFoundError:
        logging.warning(f"File {filename} not found, returning stock 0")
        return 0

# Inisialisasi
ua = UserAgent()
fake = Faker()
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

try:
    with open(os.path.join("TELEGRAM", "config.json"), "r") as config_file:
        config = json.load(config_file)
    BOT_TOKEN = config.get("bot_token")
    if not BOT_TOKEN:
        raise ValueError("Pastikan bot_token diisi dalam config.json")
except (FileNotFoundError, json.JSONDecodeError, ValueError) as e:
    print(f"❌ ERROR: {e}")
    exit(1)

ALLOWED_CHAT_ID = None
if os.path.exists(CHAT_ID_FILE):
    with open(CHAT_ID_FILE, "r", encoding="utf-8") as file:
        data = json.load(file)
        ALLOWED_CHAT_ID = data.get("chat_id")

def is_authorized(update: Update) -> bool:
    """Memeriksa apakah pengguna memiliki izin."""
    chat_id = update.effective_chat.id if update.effective_chat else None
    return chat_id == ALLOWED_CHAT_ID

async def handle_reply_keyboard(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menangani input dari Reply Keyboard pengguna."""
    text = update.message.text.strip()
    if text == "📜 DAFTAR PRODUK TERSEDIA":
        context.user_data['product_page'] = 0
        await daftar_produk(update, context)
        return
    elif text == "⬅️ Lihat produk Sebelumnya":
        if context.user_data.get('product_page', 0) > 0:
            context.user_data['product_page'] -= 1
            await daftar_produk(update, context)
        return
    elif text == "➡️ Lihat produk Berikutnya":
        context.user_data['product_page'] += 1
        await daftar_produk(update, context)
        return
    elif text == "📄 Deskripsi Produk & Harga":
        await show_description(update, context)
        return
    elif text == "👤 Admin":
        await update.message.reply_text(f"📞 Hubungi Admin: {ADMIN_URL}")
        return
    elif text == "🧾 Channel":
        await update.message.reply_text(f"📢 Channel: {CHANNEL_URL}")
        return
    elif text == "📝 TESTIMONI":
        await show_testimoni(update, context)
        return
    elif text == "📌 CARA ORDER":
        await cara_order(update, context)
        return
    elif text == "⚙️ Admin Command":
        await cmd_list(update, context)
        return
    elif text == "🔙 Kembali ke Menu Utama":
        await start(update, context)
        return

    product_mapping = context.user_data.get('product_mapping', {})
    if text in product_mapping:
        produk = product_mapping[text]
        await update.message.reply_text(f"👜 Produk {produk}", reply_markup=ReplyKeyboardRemove())
        await select_product(update, context, produk)
        return

    products = load_products()
    if text in products:
        await update.message.reply_text(f"👜 Produk {text}", reply_markup=ReplyKeyboardRemove())
        await select_product(update, context, text)
        return

async def daftar_produk(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan daftar produk dengan format baru dan tombol angka global."""
    all_products = load_products()
    if not all_products:
        await update.message.reply_text("⚠️ Saat ini tidak ada produk yang tersedia.")
        return

    current_page = context.user_data.get('product_page', 0)
    items_per_page = 15
    product_keys = list(all_products.keys())
    total_pages = (len(product_keys) - 1) // items_per_page + 1

    start_index = current_page * items_per_page
    end_index = min(start_index + items_per_page, len(product_keys))
    visible_products = product_keys[start_index:end_index]

    product_text = (
        "╭ - - - - - - - - - - - - - - - - - - - ╮\n"
        "┊  LIST PRODUK\n"
        f"┊  page {current_page + 1} / {total_pages}\n"
        "┊- - - - - - - - - - - - - - - - - - - - -\n"
    )
    context.user_data['product_mapping'] = {}
    product_buttons = []

    for index, name in enumerate(visible_products, start=start_index + 1):
        display_name = name.replace("-", " ")  # Hilangkan tanda minus untuk tampilannya saja
        product_text += f"┊ [{index}] {display_name}\n"
        product_buttons.append(KeyboardButton(str(index)))
        context.user_data['product_mapping'][str(index)] = name

    product_text += "╰ - - - - - - - - - - - - - - - - - - - ╯"

    button_layout = [product_buttons[i:i + 3] for i in range(0, len(product_buttons), 3)]
    button_layout.append([KeyboardButton("🔙 Kembali ke Menu Utama")])
    reply_markup_keyboard = ReplyKeyboardMarkup(button_layout, resize_keyboard=True)

    navigation_buttons = []
    if current_page > 0:
        navigation_buttons.append(InlineKeyboardButton("⬅️ Lihat Produk Sebelumnya", callback_data=f"prev_page_{current_page-1}"))
    if current_page < total_pages - 1:
        navigation_buttons.append(InlineKeyboardButton("➡️ Lihat Produk Berikutnya", callback_data=f"next_page_{current_page+1}"))
    reply_markup_inline = InlineKeyboardMarkup([navigation_buttons] if navigation_buttons else [])

    last_reply_message_id = context.user_data.get("last_reply_message_id")
    if update.callback_query:
        try:
            if update.callback_query.message.text != product_text:
                await update.callback_query.edit_message_text(product_text, reply_markup=reply_markup_inline)
            chat_id = update.callback_query.message.chat_id
            temp_message = await context.bot.send_message(chat_id=chat_id, text="Pilih produk dengan menekan nomor:", reply_markup=reply_markup_keyboard)
            context.user_data["last_reply_message_id"] = temp_message.message_id
            if last_reply_message_id:
                try:
                    await context.bot.delete_message(chat_id=chat_id, message_id=last_reply_message_id)
                except telegram.error.BadRequest:
                    pass
        except telegram.error.BadRequest:
            pass
    else:
        chat_id = update.message.chat_id
        sent_message = await update.message.reply_text(product_text, reply_markup=reply_markup_inline)
        temp_message = await update.message.reply_text("Pilih produk dengan menekan nomor:", reply_markup=reply_markup_keyboard)
        context.user_data["last_reply_message_id"] = temp_message.message_id
        if last_reply_message_id:
            try:
                await context.bot.delete_message(chat_id=chat_id, message_id=last_reply_message_id)
            except telegram.error.BadRequest:
                pass

async def change_page(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Mengubah halaman daftar produk."""
    query = update.callback_query
    await query.answer()
    callback_data = query.data
    if callback_data.startswith("prev_page_"):
        new_page = int(callback_data.split("_")[2])
    elif callback_data.startswith("next_page_"):
        new_page = int(callback_data.split("_")[2])
    else:
        return
    context.user_data['product_page'] = new_page
    await daftar_produk(update, context)

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan menu utama."""
    user_id = update.effective_user.id
    save_user_id(user_id)
    username = update.effective_user.username if update.effective_user.username else "Pengguna"
    bot_id = context.bot.id

    if "last_start_message" in context.user_data:
        try:
            await context.bot.delete_message(chat_id=update.effective_chat.id, message_id=context.user_data["last_start_message"])
            del context.user_data["last_start_message"]
        except Exception as e:
            print(f"⚠️ Tidak dapat menghapus pesan sebelumnya: {e}")

    if "profile_photo_message" in context.user_data:
        try:
            await context.user_data["profile_photo_message"].delete()
            del context.user_data["profile_photo_message"]
        except Exception as e:
            print(f"⚠️ Tidak dapat menghapus foto profil sebelumnya: {e}")

    update_pengunjung()
    total_pengunjung = pengunjung_data["count"]
    total_terjual = terjual_data["count"]
    waktu_sekarang = datetime.now(pytz.timezone("Asia/Jakarta")).strftime("%H:%M:%S")

    text = (
        f"✨ Hai {username}\n"
        f"Selamat datang di {STORE_NAME}\n\n"
        f"👥 Total pengunjung: {total_pengunjung}\n"
        f"📦 Total stok terjual: {total_terjual}\n"
        f"⏳ Terakhir diperbarui: {waktu_sekarang} WIB\n\n"
        "📢 Pilih menu dari tombol di bawah!"
    )

    reply_keyboard = [
        [KeyboardButton("📜 DAFTAR PRODUK TERSEDIA")],
        [KeyboardButton("👤 Admin"), KeyboardButton("🧾 Channel")],
        [KeyboardButton("📝 TESTIMONI")],
        [KeyboardButton("📌 CARA ORDER")],
        [KeyboardButton("⚙️ Admin Command")]
    ]
    reply_markup = ReplyKeyboardMarkup(reply_keyboard, resize_keyboard=True)

    bot_photos = await context.bot.get_user_profile_photos(bot_id, limit=1)
    if bot_photos and bot_photos.photos:
        try:
            profile_photo = bot_photos.photos[0][-1].file_id
            photo_message = await update.effective_message.reply_photo(profile_photo)
            context.user_data["profile_photo_message"] = photo_message
        except Exception as e:
            print(f"⚠️ Gagal mengirim foto profil: {e}")

    message = update.message or update.callback_query.message
    sent_message = await message.reply_text(text, reply_markup=reply_markup)
    context.user_data["last_start_message"] = sent_message.message_id

async def show_description(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan deskripsi produk dari Deskripsi.txt."""
    file_path = "SETUP/Deskripsi.txt"
    if not os.path.exists(file_path):
        await update.message.reply_text("⚠️ File `Deskripsi.txt` tidak ditemukan.")
        return
    try:
        with open(file_path, "r", encoding="utf-8") as file:
            description_text = file.read().strip() or "Deskripsi produk tidak tersedia."
        keyboard = [[InlineKeyboardButton("🔙 Kembali ke Menu", callback_data="start")]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        await update.message.reply_text(f"📄 *Deskripsi Produk:*\n\n{description_text}", parse_mode="Markdown", reply_markup=reply_markup)
    except Exception as e:
        await update.message.reply_text(f"❌ Terjadi kesalahan: {e}")

async def show_testimoni(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan link testimoni dari .env."""
    if TESTIMONI_URL:
        await update.message.reply_text(f"📢 *Lihat Testimoni Pengguna:*\n{TESTIMONI_URL}", parse_mode="Markdown")
    else:
        await update.message.reply_text("⚠️ Testimoni belum tersedia.")

async def cara_order(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan langkah-langkah cara order."""
    text = (
        "🛒 *CARA ORDER:*\n\n"
        "1️⃣ Buka *📜 DAFTAR PRODUK TERSEDIA*\n"
        "2️⃣ Pilih produk yang ingin dibeli\n"
        "3️⃣ Tentukan jumlah produk, lalu klik ➡️ *Beli Sekarang*\n"
        "4️⃣ Lakukan pembayaran dengan *Scan QRIS* sesuai Total Pembayaran\n"
        "5️⃣ Setelah pembayaran dikonfirmasi, akun akan dikirim otomatis dalam beberapa detik ⏳\n\n"
        "📌 *Proses cepat, otomatis, dan tanpa ribet!* 🚀"
    )
    if update.message:
        await update.message.reply_text(text, parse_mode="Markdown")
    elif update.callback_query:
        await update.callback_query.message.edit_text(text, parse_mode="Markdown")

async def select_product(update: Update, context: ContextTypes.DEFAULT_TYPE, produk: str):
    """Menampilkan detail produk setelah dipilih."""
    products = load_products()
    if produk not in products:
        await update.message.reply_text("⚠️ Produk tidak ditemukan.")
        return
    if products[produk]['stock'] <= 0:
        await update.message.reply_text(f"⚠️ Stok produk {produk} kosong dan tidak bisa dibeli.")
        return
    context.user_data['produk'] = produk
    context.user_data[f"quantity_{produk}"] = 1
    data = products[produk]
    stock_info = f"Slot Tersedia: `{data['stock']}`" if data['stock'] > 0 else "⚠️ Slot Kosong"
    description = data['desk'] if data.get('desk') else "Deskripsi tidak tersedia."
    display_name = produk.replace("-", " ")
    text = (
        f"✅ Anda memilih produk: *{display_name}*\n\n"
        f"💰 Harga: *Rp {data['price']:,}*\n"
        f"📦 {stock_info}\n"
        f"📝 Deskripsi: {description}\n\n"
        f"🔢 Jumlah: 1"
    )
    keyboard = [
    [
        InlineKeyboardButton("+", callback_data=f"increment_{produk}"),
        InlineKeyboardButton("–", callback_data=f"decrement_{produk}"),
        InlineKeyboardButton("+10", callback_data=f"increment10_{produk}")
    ],
        [InlineKeyboardButton("➡️ Beli Sekarang", callback_data=f"buy_now_{produk}")]
    ]

    reply_markup = InlineKeyboardMarkup(keyboard)
    if "selection_message" in context.user_data:
        try:
            await context.user_data["selection_message"].delete()
        except Exception as e:
            print(f"⚠️ Gagal menghapus pesan pemilihan sebelumnya: {e}")
    selection_message = await update.message.reply_text(text, parse_mode="Markdown", reply_markup=reply_markup)
    context.user_data["selection_message"] = selection_message

async def increment_decrement(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menangani tombol increment/decrement jumlah produk."""
    query = update.callback_query
    await query.answer()
    action, produk = query.data.split("_", 1)
    products = load_products()
    if produk not in products:
        await query.answer("⚠️ Produk tidak ditemukan!", show_alert=True)
        return
    quantity_key = f"quantity_{produk}"
    previous_quantity = context.user_data.get(quantity_key, 1)
    stok_tersedia = products[produk]['stock']
    if action == "increment":
        new_quantity = min(previous_quantity + 1, stok_tersedia)
    elif action == "decrement":
        new_quantity = max(previous_quantity - 1, 1)
    elif action == "increment10":
        new_quantity = min(previous_quantity + 10, stok_tersedia)
    else:
        return
    context.user_data[quantity_key] = new_quantity
    data = products[produk]
    stock_info = f"Slot Tersedia: `{data['stock']}`" if data['stock'] > 0 else "⚠️ Slot Kosong"
    description = data['desk'] if data.get('desk') else "Deskripsi tidak tersedia."
    total_harga = data['price'] * new_quantity
    text = (
        f"✅ Anda memilih produk: *{produk}*\n\n"
        f"💰 Harga: *Rp {data['price']:,}*\n"
        f"📦 {stock_info}\n"
        f"📝 Deskripsi: {description}\n\n"
        f"🔢 Jumlah: {new_quantity}\n"
        f"💵 Total Harga: Rp {total_harga:,}"
    )
    keyboard = [
    [
        InlineKeyboardButton("+", callback_data=f"increment_{produk}"),
        InlineKeyboardButton("–", callback_data=f"decrement_{produk}"),
        InlineKeyboardButton("+10", callback_data=f"increment10_{produk}")
    ],
        [InlineKeyboardButton("➡️ Beli Sekarang", callback_data=f"buy_now_{produk}")]
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    try:
        await query.edit_message_text(text, parse_mode="Markdown", reply_markup=reply_markup)
    except telegram.error.BadRequest as e:
        if "Message is not modified" in str(e):
            pass
        else:
            raise e

def load_qris_string():
    """Memuat QRIS string dari qris.json."""
    if os.path.exists(QRIS_FILE):
        with open(QRIS_FILE, "r", encoding="utf-8") as file:
            try:
                data = json.load(file)
                return data.get("qris_string", "")
            except json.JSONDecodeError:
                print("❌ ERROR: Format qris.json tidak valid.")
                return ""
    print(f"❌ ERROR: File {QRIS_FILE} tidak ditemukan.")
    return ""

QRIS_STATIC_STRING = load_qris_string()
if not QRIS_STATIC_STRING:
    print("❌ ERROR: QRIS_STATIC_STRING kosong, cek qris.json!")
    exit(1)

def to_crc16(input_str):
    """
    Menghitung CRC16-CCITT (0xFFFF) untuk string input.
    """
    crc = 0xFFFF
    for ch in input_str:
        crc ^= ord(ch) << 8
        for _ in range(8):
            if crc & 0x8000:
                crc = (crc << 1) ^ 0x1021
            else:
                crc <<= 1
            crc &= 0xFFFF
    hex_crc = hex(crc)[2:].upper().zfill(4)
    return hex_crc

def create_qris_dinamis(nominal: str, qris_string: str):
    """
    Membuat QRIS dinamis dengan nominal yang diberikan.
    
    Args:
        nominal (str): Nominal pembayaran, misalnya "10000"
        qris_string (str): QRIS statis original dari merchant

    Returns:
        io.BytesIO: Objek BytesIO berisi gambar QR code
    """
    # Hapus 4 karakter terakhir (CRC lama)
    qris2 = qris_string[:-4]

    # Ganti tag 010211 menjadi 010212 (menandakan QRIS dinamis)
    qris2 = qris2.replace("010211", "010212")

    # Pisahkan berdasarkan indikator lokasi nominal
    parts = qris2.split("5802ID")
    if len(parts) != 2:
        raise ValueError("Format QRIS tidak valid, tidak ditemukan '5802ID'")

    # Buat tag untuk nominal: 54 + panjang + nominal + 5802ID
    uang_tag = f"54{str(len(nominal)).zfill(2)}{nominal}5802ID"

    # Gabungkan kembali string QRIS dengan tag nominal di tengah
    raw_qris = parts[0] + uang_tag + parts[1]

    # Hitung CRC baru
    crc = to_crc16(raw_qris)

    # Gabungkan string akhir QRIS
    final_qris = raw_qris + crc

    # Buat QR code
    qr = qrcode.QRCode(
        version=1,
        error_correction=qrcode.constants.ERROR_CORRECT_L,
        box_size=15,
        border=4
    )
    qr.add_data(final_qris)
    qr.make(fit=True)
    img = qr.make_image(fill='black', back_color='white')
    
    # Simpan ke BytesIO
    byte_io = io.BytesIO()
    img.save(byte_io, format="PNG")
    byte_io.seek(0)
    return byte_io

async def PAY(amount):
    """Proses pembayaran dengan QRIS dinamis."""
    donation_id = str(uuid.uuid4())
    qris_string = QRIS_STATIC_STRING
    try:
        # Buat QRIS dinamis dengan nominal
        qr_image = create_qris_dinamis(str(amount), qris_string)
        return donation_id, qr_image
    except Exception as e:
        print(f"❌ Error creating QRIS: {e}")
        return None, None

async def buy_now(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menangani pembelian produk."""
    await delete_profile_photo(context)
    query = update.callback_query
    await query.answer()
    if 'donation_id' in context.user_data:
        await query.message.reply_text(
            "⚠️ Anda masih memiliki transaksi yang belum selesai!\n\n"
            "Silakan selesaikan pembayaran atau tunggu hingga transaksi sebelumnya kadaluarsa sebelum melakukan pesanan baru.",
            parse_mode="Markdown")
        return
    produk = context.user_data.get('produk', '')
    quantity = context.user_data.get(f"quantity_{produk}", 1)
    products = load_products()
    stok = get_stock_from_file(f"{produk}.txt")
    if produk not in products:
        await query.edit_message_text(f"⚠️ Produk {produk} tidak ditemukan dalam daftar.")
        return
    harga_per_akun = products[produk]['price']
    total_harga = harga_per_akun * quantity
    kode_unik = random.randint(10, 99)
    total_harga_unik = total_harga + kode_unik
    if quantity > stok:
        await query.edit_message_text(f"⚠️ Stok produk {produk} tidak mencukupi. Stok tersedia: {stok}")
        return
    for key in ["selection_message", "quantity_message", "buy_now_message"]:
        if key in context.user_data:
            try:
                await context.user_data[key].delete()
                del context.user_data[key]
            except Exception as e:
                print(f"⚠️ Gagal menghapus pesan {key}: {e}")
    donation_id, qr_image = await PAY(total_harga_unik)
    if donation_id and qr_image:
        context.user_data['donation_id'] = donation_id
        context.user_data['kode_unik'] = kode_unik
        qr_image_message = await query.message.reply_photo(InputFile(qr_image))
        context.user_data['qr_image_message'] = qr_image_message
        message = (
            f"✅ *PEMESANAN TERKONFIRMASI*\n\n"
            f"📦 Produk: {produk}\n"
            f"🆔 ID TRANSAKSI: {donation_id}\n"
            f"🔢 Jumlah: {quantity}\n"
            f"🏦 *Biaya Admin:* Rp {kode_unik}\n"
            f"💰 Total Pembayaran: Rp {total_harga_unik:,}\n\n"
            "📌 Pembayaran kadaluarsa dalam 5 menit.\n"
            "📌 Silakan scan QRIS dan pastikan nominal yang dibayarkan sesuai *Total Pembayaran* untuk menyelesaikan transaksi.")
        keyboard = [[InlineKeyboardButton("❌ BATALKAN PESANAN", callback_data=f"cancel_order_{donation_id}")]]
        reply_markup = InlineKeyboardMarkup(keyboard)
        confirmation_message = await query.message.reply_text(message, parse_mode="Markdown", reply_markup=reply_markup)
        context.user_data['confirmation_message'] = confirmation_message
        checking_message = await query.message.reply_text("⏳ Menunggu pembayaran...")
        context.user_data['checking_message'] = checking_message
        payment_task = asyncio.create_task(
            process_payment(context, query, produk, quantity, total_harga_unik, donation_id, query.from_user.id, query.from_user.username, checking_message)
        )
        context.user_data['payment_task'] = payment_task
        asyncio.create_task(delete_qris_messages_later(context))
    else:
        await query.edit_message_text("⚠️ Gagal membuat QRIS pembayaran. Silakan coba lagi atau hubungi admin.")

async def delete_profile_photo(context: ContextTypes.DEFAULT_TYPE):
    """Menghapus foto profil bot jika ada."""
    if 'profile_photo_message' in context.user_data:
        try:
            await context.user_data['profile_photo_message'].delete()
        except Exception as e:
            print(f"⚠️ Gagal menghapus foto profil: {e}")
        del context.user_data['profile_photo_message']

async def delete_qris_messages_later(context: ContextTypes.DEFAULT_TYPE, delay: int = 300):
    """Menghapus pesan QRIS setelah delay, kecuali jika sudah dibatalkan."""
    await asyncio.sleep(delay)
    if 'payment_task' in context.user_data and context.user_data['payment_task'].cancelled():
        print("✅ QRIS messages not deleted as order was cancelled.")
        return
    for key in ["qr_image_message", "confirmation_message", "checking_message"]:
        if key in context.user_data:
            try:
                await context.user_data[key].delete()
                del context.user_data[key]
                print(f"✅ Pesan {key} berhasil dihapus setelah {delay} detik.")
            except Exception as e:
                print(f"⚠️ Gagal menghapus pesan {key}: {e}")
    # Clean up user data
    context.user_data.pop('donation_id', None)
    context.user_data.pop('kode_unik', None)
    context.user_data.pop('produk', None)
    context.user_data.pop('payment_task', None)

def load_credentials():
    """Memuat kredensial dari Orkut.json."""
    if os.path.exists(ORKUT_FILE):
        with open(ORKUT_FILE, "r") as f:
            try:
                data = json.load(f)
                return data.get("MerchantCode", ""), data.get("APIKEY", "")
            except json.JSONDecodeError:
                return "", ""
    return "", ""

MERCHANT_CODE, APIKEY = load_credentials()
API_URL = f"https://gateway.okeconnect.com/api/mutasi/qris/{MERCHANT_CODE}/{APIKEY}"

def load_last_trx():
    """Memuat transaksi terakhir dari trx.json."""
    if os.path.exists(TRX_FILE):
        with open(TRX_FILE, "r") as f:
            try:
                data = json.load(f)
                return data[0] if data else None
            except json.JSONDecodeError:
                return None
    return None

def save_trx_data(data):
    """Menyimpan data transaksi ke trx.json."""
    with open(TRX_FILE, "w") as f:
        json.dump([data], f, indent=4)

def mutasiQRIS():
    """Mengambil data mutasi QRIS dari API."""
    try:
        response = httpx.get(API_URL, timeout=10)
        response.raise_for_status()
        return response.json()
    except httpx.HTTPStatusError as e:
        return {"error": f"HTTP error: {e.response.status_code}", "detail": e.response.text}
    except httpx.RequestError as e:
        return {"error": f"Request error: {str(e)}"}

async def checkpayment(target_amount):
    """Cek pembayaran setiap 5 detik selama maksimal 5 menit."""
    timeout = 300
    interval = 5
    elapsed_time = 0
    last_trx = load_last_trx()
    while elapsed_time < timeout:
        print(f"🔄 Cek pembayaran... (elapsed {elapsed_time}s)")
        hasil = mutasiQRIS()
        if hasil.get("status") == "success":
            data_mutasi = hasil.get("data", [])
            if data_mutasi:
                transaksi_terbaru = data_mutasi[0]
                date = transaksi_terbaru.get("date")
                amount = transaksi_terbaru.get("amount")
                print(f"📌 Mengecek transaksi: {date} | Amount: {amount} | Target: {target_amount}")
                if last_trx and last_trx["date"] == date and last_trx["amount"] == amount:
                    print("⏳ Belum ada transaksi baru, menunggu...")
                else:
                    if str(amount) == str(target_amount):
                        print(f"✅ Pembayaran sukses {date} sebesar {amount}")
                        save_trx_data({"date": date, "amount": amount})
                        return True
        else:
            print(f"❌ Gagal mendapatkan data: {hasil.get('error', 'Unknown error')}")
        await asyncio.sleep(interval)
        elapsed_time += interval
    print("❌ Pembayaran gagal setelah 5 menit.")
    return False

async def process_payment(context: ContextTypes.DEFAULT_TYPE, query, produk: str, jumlah: int, total_harga: int, donation_id: str, user_id: int, username: str, checking_message):
    """Memproses pembayaran dan mengirim akun."""
    try:
        pembayaran_berhasil = await checkpayment(total_harga)
        if pembayaran_berhasil:
            try:
                produk_file = f"{produk}.txt"
                with open(produk_file, "r", encoding="utf-8") as file:
                    akun_lines = [line.strip() for line in file if line.strip()]
                stok_tersedia = len(akun_lines)
                if stok_tersedia < jumlah:
                    await query.message.reply_text(
                        f"❌ Maaf, stok akun *{produk}* hanya tersedia {stok_tersedia}, "
                        f"sedangkan pesanan Anda {jumlah} akun.\n"
                        "Mohon hubungi admin untuk refund atau solusi lainnya.",
                        parse_mode="Markdown")
                    await notify_admin_payment(context, user_id, username, produk, stok_tersedia, total_harga, donation_id)
                    return
                akun_terkirim = akun_lines[:jumlah]
                sisa_akun = akun_lines[jumlah:]
                with open(produk_file, "w", encoding="utf-8") as file:
                    file.write("\n".join(sisa_akun))
                # Update stok
                stock_data = load_initial_stock()
                stock_data[produk] = len(sisa_akun)
                save_initial_stock(stock_data)
                update_terjual(jumlah)
                akun_message = "\n".join(akun_terkirim)
                # 1. Simpan akun ke file
                filename = f"{donation_id}.txt"
                file_path = os.path.join("TRANSAKSI", filename)
                os.makedirs("TRANSAKSI", exist_ok=True)
                with open(file_path, "w", encoding="utf-8") as f:
                    f.write(akun_message)

                # 2. Kirim file ke user
                with open(file_path, "rb") as f:
                    await query.message.reply_document(
                        document=InputFile(f, filename=filename),
                        caption=f"✅ *Akun berhasil dibeli*\n🆔 ID: `{donation_id}`",
                        parse_mode="Markdown"
                    )
                waktu_pembayaran = datetime.now(pytz.timezone("Asia/Jakarta")).strftime("%d-%m-%Y %H:%M:%S")
                message = (
                    f"✅ *PEMBAYARAN BERHASIL*\n\n"
                    f"📌 Produk: {produk}\n"
                    f"🆔 ID TRANSAKSI: {donation_id}\n"
                    f"🔢 Jumlah: {jumlah}\n"
                    f"⏰ Waktu: {waktu_pembayaran}\n\n"
                    f"📃 Silahkan cek file .txt untuk melihat akun anda")
                await delete_previous_messages(context)
                await query.message.reply_text(message, parse_mode="Markdown")
                await notify_admin_payment(context, user_id, username, produk, jumlah, total_harga, donation_id)
            except FileNotFoundError:
                await query.message.reply_text(f"❌ File {produk}.txt tidak ditemukan.")
            except Exception as e:
                await query.message.reply_text(f"❌ Terjadi kesalahan: {e}")
        else:
            await delete_previous_messages(context)
            await query.message.reply_text(
                "⚠️ *Waktu pembayaran telah habis!*\n\n"
                "Anda tidak menyelesaikan pembayaran dalam 5 menit.\n"
                "Silakan lakukan *order ulang* jika masih ingin membeli produk ini.\n\n"
                "📌 *Pastikan Anda membayar tepat waktu untuk menghindari kegagalan transaksi!*",
                parse_mode="Markdown")
    except asyncio.CancelledError:
        print(f"✅ Payment processing for donation_id {donation_id} was cancelled.")
        raise
    finally:
        context.user_data.pop('donation_id', None)
        context.user_data.pop('kode_unik', None)
        context.user_data.pop('produk', None)
        context.user_data.pop('checking_message', None)
        context.user_data.pop('payment_task', None)

async def delete_previous_messages(context):
    """Menghapus pesan sebelumnya terkait pembayaran."""
    messages_to_delete = ["selection_message", "qr_image_message", "confirmation_message", "donepay_button_message"]
    for key in messages_to_delete:
        if key in context.user_data:
            try:
                await context.user_data[key].delete()
                del context.user_data[key]
            except Exception as e:
                print(f"Error deleting {key}: {e}")

async def notify_admin_payment(context, user_id, username, produk, jumlah, total_harga, donation_id):
    """Mengirim notifikasi ke admin saat pembayaran berhasil menggunakan bot lain."""
    # ALLOWED_CHAT_ID di sini haruslah chat_id admin yang akan menerima notifikasi dari BOT KEDUA
    if not ALLOWED_CHAT_ID:
        print("⚠️ Chat ID admin (untuk bot notifikasi) tidak ditemukan, notifikasi tidak dikirim.")
        return
    if not ADMIN_BOT: # Pastikan ADMIN_BOT sudah diinisialisasi
        print("❌ Admin Notification Bot tidak aktif. Notifikasi admin tidak dikirim.")
        return

    waktu_pembayaran = datetime.now(pytz.timezone("Asia/Jakarta")).strftime("%d-%m-%Y %H:%M:%S")
    username_display = f"@{username}" if username else f"`{user_id}`"
    message = (
        f"📢 *PEMBAYARAN BERHASIL! (Notifikasi Otomatis)*\n\n"
        f"👤 *Pembeli:* {username_display}\n"
        f"📦 *Produk:* {produk}\n"
        f"🔢 *Jumlah:* {jumlah}\n"
        f"💰 *Total Bayar:* Rp {total_harga:,}\n"
        f"🆔 *ID Transaksi:* {donation_id}\n"
        f"⏰ *Waktu:* {waktu_pembayaran} WIB")
    try:
        # Menggunakan ADMIN_BOT untuk mengirim pesan
        await ADMIN_BOT.send_message(chat_id=ALLOWED_CHAT_ID, text=message, parse_mode="Markdown")
        print(f"✅ Notifikasi berhasil dikirim ke admin {ALLOWED_CHAT_ID} via bot notifikasi.")
    except Exception as e:
        print(f"❌ Gagal mengirim notifikasi admin via bot notifikasi: {e}")

async def cancel_order(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menangani pembatalan pesanan dan menghentikan pengecekan pembayaran."""
    query = update.callback_query
    await query.answer()
    donation_id = query.data.split("_", 2)[2]
    if donation_id != context.user_data.get('donation_id'):
        await query.message.reply_text("⚠️ Transaksi tidak valid atau sudah kadaluarsa.", parse_mode="Markdown")
        return
    # Cancel the payment task if it exists
    payment_task = context.user_data.get('payment_task')
    if payment_task and not payment_task.done():
        payment_task.cancel()
        try:
            await payment_task
        except asyncio.CancelledError:
            print(f"✅ Payment task for donation_id {donation_id} cancelled.")
    # Delete related messages
    messages_to_delete = ["qr_image_message", "confirmation_message", "checking_message"]
    for key in messages_to_delete:
        if key in context.user_data:
            try:
                await context.user_data[key].delete()
                del context.user_data[key]
            except Exception as e:
                print(f"⚠️ Gagal menghapus pesan {key}: {e}")
    # Clean up user data
    context.user_data.pop('donation_id', None)
    context.user_data.pop('kode_unik', None)
    context.user_data.pop('produk', None)
    context.user_data.pop('payment_task', None)
    context.user_data.pop('quantity_' + context.user_data.get('produk', ''), None)
    # Notify user
    await query.message.reply_text(
        "❌ *Pesanan Dibatalkan*\n\n"
        "Pesanan Anda telah dibatalkan dan pengecekan pembayaran dihentikan.\n"
        "Silakan mulai pesanan baru jika diperlukan.", parse_mode="Markdown"
    )
    await start(update, context)

async def add_account(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan daftar file .txt untuk ditambahkan akun."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return
    txt_files = [f for f in os.listdir() if f.endswith(".txt")]
    if not txt_files:
        await update.message.reply_text("⚠️ Tidak ada file .txt yang tersedia.")
        return
    keyboard = [[InlineKeyboardButton(file, callback_data=f"addfile_{file}")] for file in txt_files]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text("📂 Pilih file untuk menambahkan akun:", reply_markup=reply_markup)

async def select_add_file(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menangani pemilihan file untuk menambahkan akun."""
    query = update.callback_query
    await query.answer()
    filename = query.data.split("_", 1)[1]
    context.user_data['selected_file'] = filename
    context.user_data['adding_account'] = True
    await query.edit_message_text(
        f"✅ Anda memilih file: `{filename}`\n\nSilakan kirim akun yang ingin ditambahkan.\n\n"
        "Contoh format:\n`email|password`", parse_mode="Markdown")

async def receive_account(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menyimpan akun ke file yang dipilih."""
    if not context.user_data.get('adding_account', False):
        logging.warning(f"receive_account triggered without adding_account for user {update.effective_user.id}")
        return

    file_path = context.user_data.get('selected_file')
    # Memisahkan akun berdasarkan baris baru dan membersihkan spasi kosong
    new_accounts = [line.strip() for line in update.message.text.split('\n') if line.strip()]

    if not file_path or not new_accounts:
        await update.message.reply_text("⚠️ Format akun salah atau tidak ada file yang dipilih atau akun yang valid.")
        return

    accounts_added_count = 0
    try:
        # Hitung jumlah akun sebelum penambahan (opsional, untuk tampilan total)
        initial_account_count = 0
        try:
            with open(file_path, "r", encoding="utf-8") as file:
                initial_account_count = len([line for line in file if line.strip()])
        except FileNotFoundError:
            pass # File doesn't exist yet, so initial count is 0

        with open(file_path, "a", encoding="utf-8") as file:
            for account in new_accounts:
                file.write(f"\n{account}")
                accounts_added_count += 1

        # Hitung jumlah akun setelah penambahan
        with open(file_path, "r", encoding="utf-8") as file:
            final_account_count = len([line for line in file if line.strip()])

        await update.message.reply_text(
            f"✅ Berhasil menambahkan **{accounts_added_count}** akun ke `{file_path}`!\n"
            f"📈 Total akun sekarang: {final_account_count} akun."
        )
        context.user_data.pop('adding_account', None)
        context.user_data.pop('selected_file', None)
        
        # Perbarui initial_stock.json untuk mencerminkan stok baru
        stock_data = load_initial_stock()
        product_name = os.path.splitext(os.path.basename(file_path))[0]
        stock_data[product_name] = get_stock_from_file(file_path)
        save_initial_stock(stock_data)

    except Exception as e:
        await update.message.reply_text(f"❌ Terjadi kesalahan: {e}")

async def create_txt_file(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Membuat file .txt baru."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return
    if not context.args or len(context.args) != 1:
        await update.message.reply_text("⚠️ Format salah! Gunakan:\n\n`/createtxt nama_file.txt`", parse_mode="Markdown")
        return
    filename = context.args[0]
    if not filename.endswith(".txt"):
        await update.message.reply_text("❌ Nama file harus mengandung `.txt`!\nContoh yang benar:\n`/createtxt Zoom.txt`", parse_mode="Markdown")
        return
    file_path = os.path.join(os.getcwd(), filename)
    if os.path.exists(file_path):
        await update.message.reply_text(f"⚠️ File `{filename}` sudah ada!", parse_mode="Markdown")
        return
    try:
        with open(file_path, "w", encoding="utf-8") as file:
            file.write("")
        await update.message.reply_text(f"✅ File `{filename}` berhasil dibuat!", parse_mode="Markdown")
        # Tambahkan ke initial_stock.json dengan stok 0
        stock_data = load_initial_stock()
        product_name = os.path.splitext(os.path.basename(filename))[0]
        stock_data[product_name] = 0
        save_initial_stock(stock_data)
    except Exception as e:
        await update.message.reply_text(f"❌ Gagal membuat file: {str(e)}")

# ConversationHandler for /edits
EDIT_FILE, RECEIVE_CONTENT = range(2)

async def start_edit(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Memulai proses pengeditan file."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return ConversationHandler.END
    if not context.args:
        await update.message.reply_text(
            "⚠️ Format salah! Gunakan:\n\n"
            "`/edits namaproduk.txt`\n"
            "`/edits PRODUCTS/Ready.txt`", parse_mode="Markdown")
        return ConversationHandler.END
    filename = context.args[0]
    file_path = os.path.join(os.getcwd(), filename)
    if not os.path.exists(file_path):
        await update.message.reply_text(f"❌ File `{filename}` tidak ditemukan.", parse_mode="Markdown")
        return ConversationHandler.END
    try:
        with open(file_path, "r", encoding="utf-8") as file:
            old_content = file.read() or "(File kosong)"
        await update.message.reply_text(
            f"📄 Isi Saat Ini dari `{filename}`:\n\n{old_content}\n\n"
            "Kirim teks baru untuk menggantinya.\n\n"
            "Gunakan `/cancel` untuk membatalkan.", parse_mode="Markdown")
        context.user_data['edit_file'] = file_path
        return RECEIVE_CONTENT
    except Exception as e:
        await update.message.reply_text(f"❌ Terjadi kesalahan: {e}", parse_mode="Markdown")
        return ConversationHandler.END

async def save_edited_content(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menyimpan teks baru ke file yang diedit."""
    file_path = context.user_data.get('edit_file')
    if not file_path:
        logging.warning(f"save_edited_content triggered without edit_file for user {update.effective_user.id}")
        await update.message.reply_text("❌ Sesi pengeditan tidak ditemukan. Gunakan `/edits` untuk memulai.", parse_mode="Markdown")
        return ConversationHandler.END
    new_content = update.message.text
    try:
        with open(file_path, "w", encoding="utf-8") as file:
            file.write(new_content)
        await update.message.reply_text(f"✅ `{os.path.basename(file_path)}` berhasil diperbarui!", parse_mode="Markdown")
        # Perbarui initial_stock.json jika file adalah file produk
        if not file_path.endswith("Deskripsi.txt") and not file_path.endswith("Ready.txt"):
            stock_data = load_initial_stock()
            product_name = os.path.splitext(os.path.basename(file_path))[0]
            stock_data[product_name] = len([line for line in new_content.splitlines() if line.strip()])
            save_initial_stock(stock_data)
        context.user_data.pop('edit_file', None)
        return ConversationHandler.END
    except Exception as e:
        await update.message.reply_text(f"❌ Terjadi kesalahan: {e}", parse_mode="Markdown")
        return ConversationHandler.END

async def cancel_edit(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Membatalkan proses pengeditan."""
    context.user_data.pop('edit_file', None)
    await update.message.reply_text("❌ Pengeditan dibatalkan.", parse_mode="Markdown")
    return ConversationHandler.END

def is_protected_file(file_path):
    """Memeriksa apakah file dilindungi."""
    protected_files = {
        os.path.abspath("PRODUCTS/Ready.txt"),
        os.path.abspath("SETUP/Deskripsi.txt")
    }
    return os.path.abspath(file_path) in protected_files

async def delete_txt_file(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan daftar file .txt untuk dihapus."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return
    txt_files = []
    for root, _, files in os.walk(os.getcwd()):
        for file in files:
            if file.endswith(".txt"):
                relative_path = os.path.relpath(os.path.join(root, file))
                txt_files.append(relative_path)
    if not txt_files:
        await update.message.reply_text("⚠️ Tidak ada file .txt yang tersedia untuk dihapus.")
        return
    keyboard = [[InlineKeyboardButton(file, callback_data=f"delete_{file}")] for file in txt_files]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text("🗑 Pilih file yang ingin dihapus:", reply_markup=reply_markup)

async def delete_selected_txt(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menghapus file yang dipilih."""
    query = update.callback_query
    await query.answer()
    filename = query.data.split("_", 1)[1]
    file_path = os.path.abspath(os.path.join(os.getcwd(), filename))
    if not file_path.startswith(os.getcwd()):
        await query.edit_message_text(f"⚠️ Akses ditolak! Tidak dapat menghapus `{filename}`.", parse_mode="Markdown")
        return
    if is_protected_file(file_path):
        await query.edit_message_text(f"⚠️ File `{filename}` tidak bisa dihapus karena file ini penting!", parse_mode="Markdown")
        return
    if not os.path.exists(file_path):
        await query.edit_message_text(f"❌ File `{filename}` tidak ditemukan.", parse_mode="Markdown")
        return
    try:
        os.remove(file_path)
        await query.edit_message_text(f"✅ File `{filename}` berhasil dihapus!", parse_mode="Markdown")
        # Hapus dari initial_stock.json jika file adalah file produk
        stock_data = load_initial_stock()
        product_name = os.path.splitext(os.path.basename(filename))[0]
        if product_name in stock_data:
            del stock_data[product_name]
            save_initial_stock(stock_data)
    except Exception as e:
        await query.edit_message_text(f"❌ Terjadi kesalahan: {e}", parse_mode="Markdown")

async def see_txt_files(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan daftar file .txt atau isi file."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return
    if context.args:
        filename = context.args[0]
        file_path = os.path.join(os.getcwd(), filename)
        if not os.path.exists(file_path):
            await update.message.reply_text(f"❌ File `{filename}` tidak ditemukan.", parse_mode="Markdown")
            return
        try:
            with open(file_path, "r", encoding="utf-8") as file:
                content = file.read() or "(File kosong)"
            await update.message.reply_text(f"📄 Isi `{filename}`:\n\n{content}", parse_mode="Markdown")
        except Exception as e:
            await update.message.reply_text(f"❌ Terjadi kesalahan: {e}", parse_mode="Markdown")
    else:
        txt_files = []
        for root, _, files in os.walk(os.getcwd()):
            for file in files:
                if file.endswith(".txt"):
                    relative_path = os.path.relpath(os.path.join(root, file))
                    txt_files.append(relative_path)
        if not txt_files:
            await update.message.reply_text("⚠️ Tidak ada file .txt yang tersedia.")
            return
        keyboard = [[InlineKeyboardButton(file, callback_data=f"view_{file}")] for file in txt_files]
        reply_markup = InlineKeyboardMarkup(keyboard)
        await update.message.reply_text("📂 Pilih file untuk melihat isinya:", reply_markup=reply_markup)

async def view_txt_content(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan isi file yang dipilih."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return
    query = update.callback_query
    await query.answer()
    filename = query.data.split("_")[1]
    file_path = os.path.join(os.getcwd(), filename)
    if not os.path.exists(file_path):
        await query.edit_message_text(f"❌ File `{filename}` tidak ditemukan.", parse_mode="Markdown")
        return
    try:
        with open(file_path, "r", encoding="utf-8") as file:
            content = file.read() or "(File kosong)"
        await query.edit_message_text(f"📄 Isi `{filename}`:\n\n{content}", parse_mode="Markdown")
    except Exception as e:
        await query.edit_message_text(f"❌ Terjadi kesalahan: {e}", parse_mode="Markdown")

async def add_desk(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menambahkan atau memperbarui deskripsi produk."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return
    if len(context.args) < 2:
        await update.message.reply_text("⚠️ Format salah! Gunakan:\n\n`/desk namaproduk deskripsi_produk`", parse_mode="Markdown")
        return
    name = context.args[0]
    description = " ".join(context.args[1:])
    all_products = load_products()
    if name not in all_products:
        await update.message.reply_text(f"❌ Produk `{name}` tidak ditemukan dalam daftar! Pastikan nama produk sudah benar.", parse_mode="Markdown")
        return
    desk_data = load_desk()
    if name in desk_data:
        old_description = desk_data[name]
        desk_data[name] = description
        await update.message.reply_text(
            f"🔄 Deskripsi untuk `{name}` telah diperbarui!\n\n"
            f"*Sebelumnya:* {old_description}\n"
            f"*Sekarang:* {description}", parse_mode="Markdown")
    else:
        desk_data[name] = description
        await update.message.reply_text(f"✅ Deskripsi untuk `{name}` telah ditambahkan!", parse_mode="Markdown")
    try:
        with open("desk.json", "w", encoding="utf-8") as file:
            json.dump(desk_data, file, indent=4)
    except Exception as e:
        await update.message.reply_text(f"❌ Gagal menyimpan deskripsi: {str(e)}", parse_mode="Markdown")

async def notify(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Memulai proses broadcast (teks atau gambar + teks)."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return

    # Cek jika ada media
    if update.message.photo:
        # Ambil caption jika ada
        caption = update.message.caption or "📢 Pesan tanpa teks"
        file_id = update.message.photo[-1].file_id
        await update.message.reply_text("📨 Sedang mengirim gambar ke semua pengguna...")
        asyncio.create_task(send_notifications_with_photo(context, file_id, caption, update.effective_chat.id))
    elif update.message.text:
        full_message_text = update.message.text
        try:
            command_end_index = full_message_text.index(' ')
            broadcast_message = full_message_text[command_end_index + 1:].strip()
        except ValueError:
            broadcast_message = ""
        if not broadcast_message:
            await update.message.reply_text("⚠️ Harap masukkan pesan. Contoh:\n/notify Halo!", parse_mode="Markdown")
            return
        await update.message.reply_text(f"📨 Sedang mengirim pesan ke semua pengguna...")
        asyncio.create_task(send_notifications(context, broadcast_message, update.effective_chat.id))
    else:
        await update.message.reply_text("⚠️ Kirim teks atau gambar dengan caption untuk broadcast.")

# Fungsi send_notifications tidak perlu diubah karena sudah menerima 'message' sebagai string
async def send_notifications(context: ContextTypes.DEFAULT_TYPE, message: str, admin_chat_id: int):
    """Mengirim broadcast ke semua pengguna."""
    users_list = load_users_list()
    success_count = 0
    failed_count = 0
    for user_id in users_list:
        try:
            # Di sini, 'message' sudah berisi line break jika ada
            await context.bot.send_message(chat_id=user_id, text=f"📢 *BROADCAST MESSAGE:*\n\n{message}", parse_mode="Markdown")
            success_count += 1
        except Exception as e:
            print(f"⚠️ Gagal mengirim pesan ke {user_id}: {e}")
            failed_count += 1
    await context.bot.send_message(chat_id=admin_chat_id, text=f"✅ Broadcast selesai!\n\n📨 Terkirim: {success_count}\n❌ Gagal: {failed_count}", parse_mode="Markdown")

async def send_notifications_with_photo(context: ContextTypes.DEFAULT_TYPE, file_id: str, caption: str, admin_chat_id: int):
    """Mengirim broadcast gambar + caption ke semua user."""
    users_list = load_users_list()
    success_count = 0
    failed_count = 0
    for user_id in users_list:
        try:
            await context.bot.send_photo(chat_id=user_id, photo=file_id, caption=caption, parse_mode="Markdown")
            success_count += 1
        except Exception as e:
            print(f"❌ Gagal kirim ke {user_id}: {e}")
            failed_count += 1
    await context.bot.send_message(chat_id=admin_chat_id, text=f"✅ Broadcast gambar selesai!\n📤 Terkirim: {success_count}\n❌ Gagal: {failed_count}")

def load_users_list():
    """Memuat daftar pengguna dari users.json."""
    try:
        if os.path.exists(USERS_FILE):
            with open(USERS_FILE, "r", encoding="utf-8") as file:
                data = json.load(file)
                if isinstance(data, list):
                    return data  # Return list directly if that's the format
                elif isinstance(data, dict):
                    return data.get("users", [])  # Return users list from dict
                else:
                    logging.error(f"Invalid format in users.json: Expected list or dict, got {type(data)}")
                    return []
        return []
    except Exception as e:
        logging.error(f"Error loading users.json: {e}")
        return []
    
def save_user_id(user_id: int):
    """Menyimpan chat_id pengguna ke users.json secara otomatis."""
    try:
        if os.path.exists(USERS_FILE):
            with open(USERS_FILE, "r", encoding="utf-8") as f:
                data = json.load(f)
        else:
            data = []

        if isinstance(data, dict):
            users = data.get("users", [])
            if user_id not in users:
                users.append(user_id)
                data["users"] = users
        elif isinstance(data, list):
            if user_id not in data:
                data.append(user_id)
        else:
            data = [user_id]

        with open(USERS_FILE, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=4)
        print(f"✅ chat_id {user_id} disimpan ke users.json")
    except Exception as e:
        print(f"❌ Gagal menyimpan chat_id: {e}")

async def cmd_list(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan daftar perintah admin."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return
    keyboard = [
        [InlineKeyboardButton("➕ Tambah Akun", callback_data="cmd_add")],
        [InlineKeyboardButton("🗑 Hapus File", callback_data="cmd_deletetxt")],
        [InlineKeyboardButton("📂 Buat File Baru", callback_data="cmd_createtxt")],
        [InlineKeyboardButton("📢 Kirim Broadcast", callback_data="cmd_notify")],
        [InlineKeyboardButton("🔄 Lihat File", callback_data="cmd_see")],
        [InlineKeyboardButton("✏️ EDIT MANAGER", callback_data="cmd_edits")],
        [InlineKeyboardButton("📝 TAMBAHKAN DESK", callback_data="cmd_tambahdesk")],
        [InlineKeyboardButton("⬅️ Kembali", callback_data="start")]
    ]
    text = (
        "📜 *Daftar Perintah Bot:*\n\n"
        "➕ *Tambah Akun:* Menambahkan akun ke file .txt\n"
        "🗑 *Hapus File:* Menghapus file .txt yang dipilih\n"
        "📂 *Buat File Baru:* Membuat file .txt baru\n"
        "📢 *Kirim Broadcast:* Mengirim pesan ke semua pengguna\n"
        "🔄 *Lihat File:* Menampilkan daftar file .txt\n"
        "✏️ *EDIT MANAGER:* Memilih file untuk diedit\n"
        "📝 *TAMBAHKAN DESK:* Tambah atau edit deskripsi produk\n")
    if update.message:
        await update.message.reply_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode="Markdown")
    elif update.callback_query:
        await update.callback_query.edit_message_text(text, reply_markup=InlineKeyboardMarkup(keyboard), parse_mode="Markdown")

async def edit_manager(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menampilkan format penggunaan /edits."""
    if not is_authorized(update):
        await restricted_access(update, context)
        return
    message_text = (
        "⚠️ Format salah! Gunakan:\n"
        "`/edits namaproduk.txt`\n"
        "`/edits PRODUCTS/Ready.txt`")
    if update.message:
        await update.message.reply_text(message_text, parse_mode="Markdown")
    elif update.callback_query:
        await update.callback_query.edit_message_text(message_text, parse_mode="Markdown")

async def restricted_access(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Mengirim pesan akses terbatas."""
    message_text = "⚠️ Akses terbatas untuk admin."
    if update.message:
        await update.message.reply_text(message_text)
    elif update.callback_query:
        await update.callback_query.answer(message_text, show_alert=True)

async def handle_command_buttons(update: Update, context: ContextTypes.DEFAULT_TYPE):
    """Menangani tombol perintah admin."""
    query = update.callback_query
    await query.answer()
    if query.data == "cmd_tambahdesk":
        await query.message.reply_text("⚠️ Format salah! Gunakan:\n\n`/desk namaproduk deskripsi_produk`", parse_mode="Markdown")
        return
    if query.data == "cmd":
        await cmd_list(update, context)
        return
    command = query.data.replace("cmd_", "/")
    try:
        await query.message.delete()
    except Exception as e:
        print(f"⚠️ Tidak dapat menghapus pesan: {e}")
    fake_update = Update(update_id=update.update_id, message=query.message)
    if command == "/add":
        await add_account(fake_update, context)
    elif command == "/deletetxt":
        await delete_txt_file(fake_update, context)
    elif command == "/createtxt":
        await create_txt_file(fake_update, context)
    elif command == "/notify":
        await notify(fake_update, context)
    elif command == "/see":
        await see_txt_files(fake_update, context)
    elif command == "/edits":
        await start_edit(fake_update, context)
    elif command == "/start":
        await start(fake_update, context)

def main():
    """Fungsi utama untuk menjalankan bot."""
    threading.Thread(target=run_flask, daemon=True).start()
    threading.Thread(target=start_keep_alive, daemon=True).start()
    application = Application.builder().token(BOT_TOKEN).build()

    # ConversationHandler for /edits
    edit_conv_handler = ConversationHandler(
        entry_points=[CommandHandler("edits", start_edit)],
        states={
            RECEIVE_CONTENT: [MessageHandler(filters.TEXT & ~filters.COMMAND, save_edited_content)],
        },
        fallbacks=[CommandHandler("cancel", cancel_edit)],
    )

    # ConversationHandler for take stock by admin
    take_stock_conv_handler = ConversationHandler(
        entry_points=[CommandHandler("ambilstok", take_stock_command)],
        states={
            TAKE_STOCK_PRODUCT: [CallbackQueryHandler(take_stock_select_product, pattern=r"^takestock_")],
            TAKE_STOCK_QUANTITY: [MessageHandler(filters.TEXT & ~filters.COMMAND, take_stock_enter_quantity)],
        },
        fallbacks=[CommandHandler("cancel", cancel_take_stock)],
    )

    # Handlers
    # ... (handlers yang sudah ada) ...
    application.add_handler(take_stock_conv_handler) # Tambahkan ini
    application.add_handler(CommandHandler("ambilstok", take_stock_command)) # Tambahkan ini untuk entry point awal

    # Tambahkan tombol ke menu admin cmd_list
    # Di dalam fungsi cmd_list, tambahkan baris berikut ke keyboard:
    # [InlineKeyboardButton("📦 Ambil Stok", callback_data="cmd_ambilstok")]
    # Dan tambahkan handler di handle_command_buttons:
    # elif command == "/ambilstok":
    #     await take_stock_command(fake_update, context)

    # Handlers
    application.add_handler(CallbackQueryHandler(change_page, pattern="^(prev_page|next_page)$"))
    application.add_handler(MessageHandler(filters.TEXT & filters.Regex(r"^⬅️ Lihat Produk Sebelumnya$"), change_page))
    application.add_handler(MessageHandler(filters.TEXT & filters.Regex(r"^➡️ Lihat Produk Berikutnya$"), change_page))
    application.add_handler(CallbackQueryHandler(increment_decrement, pattern="^(increment|decrement|increment10)_"))
    application.add_handler(CommandHandler("daftarproduk", daftar_produk))
    application.add_handler(CallbackQueryHandler(change_page, pattern=r"^(prev_page|next_page)_\d+$"))
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_reply_keyboard), group=3)
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, receive_account), group=2)
    application.add_handler(edit_conv_handler)
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("deletetxt", delete_txt_file))
    application.add_handler(CommandHandler("add", add_account))
    application.add_handler(CommandHandler("createtxt", create_txt_file))
    application.add_handler(CommandHandler("cmd", cmd_list))
    application.add_handler(CommandHandler("notify", notify))
    application.add_handler(CommandHandler("see", see_txt_files))
    application.add_handler(CallbackQueryHandler(edit_manager, pattern=r"^cmd_edits$"))
    application.add_handler(CallbackQueryHandler(delete_selected_txt, pattern=r"^delete_"))
    application.add_handler(CallbackQueryHandler(buy_now, pattern=r"^buy_now_"))
    application.add_handler(CallbackQueryHandler(select_add_file, pattern=r"^addfile_"))
    application.add_handler(CallbackQueryHandler(view_txt_content, pattern=r"^view_"))
    application.add_handler(CallbackQueryHandler(cancel_order, pattern=r"^cancel_order_"))
    application.add_handler(CommandHandler("desk", add_desk))
    application.add_handler(CallbackQueryHandler(handle_command_buttons))

    application.run_polling()
    
if __name__ == '__main__':
    main()
