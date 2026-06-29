import asyncio
import logging
import os
import re
import json
import time
import glob
from aiogram import Bot, Dispatcher, types, F, Router
from aiogram.types import InlineQueryResultCachedVideo, InlineQueryResultCachedPhoto
from aiogram.filters import Command, StateFilter
from aiogram.types import FSInputFile, InlineKeyboardButton, InputMediaPhoto, InputMediaVideo, ReplyKeyboardMarkup, \
    KeyboardButton
from aiogram.enums import ParseMode
from aiogram.client.default import DefaultBotProperties
from aiogram.utils.keyboard import InlineKeyboardBuilder
from aiogram.fsm.state import StatesGroup, State
from aiogram.fsm.context import FSMContext
import yt_dlp

# --- 1. НАСТРОЙКИ И КОНФИГУРАЦИЯ ---
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

TOKEN = "8985816100:AAHt2ATi23Fr-S1c8bUljd8X2nyVrC1YwJg"
ADMIN_ID = 7661308568

bot = Bot(token=TOKEN, default=DefaultBotProperties(parse_mode=ParseMode.HTML))
dp = Dispatcher()
router = Router()
dp.include_router(router)

DOWNLOAD_DIR = "downloads"
os.makedirs(DOWNLOAD_DIR, exist_ok=True)

COOKIES_FILE = "instagram_cookies.txt"  # куки для Instagram

URL_PATTERN = re.compile(
    r'https?://(?:www\.)?[-a-zA-Z0-9@:%._\+~#=]{1,256}\.[a-zA-Z0-9()]{1,6}\b(?:[-a-zA-Z0-9()@:%_\+.~#?&//=]*)')

# Время первого нажатия «Продолжить» для каждого юзера
sub_check_times = {}  # user_id -> timestamp

# Последние скачанные file_id для инлайн-режима: user_id -> [{"file_id": ..., "type": "video"/"photo"}, ...]
user_last_media = {}  # user_id -> list

# Поддерживаемые расширения
VIDEO_EXTENSIONS = {'.mp4', '.mov', '.avi', '.mkv', '.webm', '.flv', '.m4v'}
PHOTO_EXTENSIONS = {'.jpg', '.jpeg', '.png', '.webp', '.gif'}


class AdminStates(StatesGroup):
    waiting_for_broadcast = State()
    waiting_for_new_channel = State()
    waiting_for_new_caption = State()


# --- 2. БАЗА ДАННЫХ ---
DB_FILE = 'bot_db.json'


def save_db():
    try:
        with open(DB_FILE, 'w', encoding='utf-8') as f:
            json.dump(db, f, ensure_ascii=False, indent=4)
    except Exception as e:
        logger.error(f"Ошибка сохранения базы данных: {e}")


def sanitize_channels(raw_channels):
    clean = []
    dropped = 0
    if isinstance(raw_channels, list):
        for item in raw_channels:
            if isinstance(item, dict) and item.get("name") and item.get("url"):
                clean.append({"name": str(item["name"]), "url": str(item["url"])})
            else:
                dropped += 1
    if dropped:
        logger.warning(f"⚠️ Пропущено {dropped} некорректных записей в channels")
    return clean


def sanitize_captions(raw_captions):
    clean = []
    dropped = 0
    if isinstance(raw_captions, list):
        for item in raw_captions:
            if isinstance(item, dict) and item.get("text"):
                clean.append({
                    "name": str(item.get("name") or f"Описание {len(clean) + 1}"),
                    "text": str(item["text"]),
                    "buttons": item.get("buttons") if isinstance(item.get("buttons"), list) else []
                })
            else:
                dropped += 1
    if dropped:
        logger.warning(f"⚠️ Пропущено {dropped} некорректных записей в captions")
    return clean


def load_db():
    default = {
        "approved_users": [],
        "all_users": [],
        "banned_users": [],
        "captions": [
            {
                "name": "Стандартное рекламное",
                "text": '🔥 Загляни в наш основной канал, там много интересного контента!',
                "buttons": []
            }
        ],
        "active_caption_index": 0,
        "send_as_first_message": False,  # Реклама Изначально ВЫКЛЮЧЕНА
        "channels": [
            {"name": "Канал №1 🎯", "url": "https://t.me/+U8L1D1ALWLU5ODky"},
            {"name": "Канал №2 🚀", "url": "https://t.me/+XbT6PCkGO801ZjAy"},
        ]
    }
    if os.path.exists(DB_FILE):
        try:
            with open(DB_FILE, 'r', encoding='utf-8') as f:
                data = json.load(f)
                if "approved_users" not in data: data["approved_users"] = []
                if "all_users" not in data: data["all_users"] = []
                if "banned_users" not in data: data["banned_users"] = []
                if "channels" not in data: data["channels"] = default["channels"]
                data["channels"] = sanitize_channels(data["channels"])
                if not data["channels"]:
                    data["channels"] = default["channels"]

                if "captions" not in data:
                    data["captions"] = default["captions"]
                data["captions"] = sanitize_captions(data["captions"])
                if not data["captions"]:
                    data["captions"] = default["captions"]

                if "active_caption_index" not in data or not isinstance(data["active_caption_index"], int) \
                        or not (0 <= data["active_caption_index"] < len(data["captions"])):
                    data["active_caption_index"] = 0

                if "send_as_first_message" not in data:
                    data["send_as_first_message"] = False

                return data
        except Exception:
            pass
    return default


db = load_db()
approved_users = set(db["approved_users"])
all_users = set(db["all_users"])
banned_users = set(db["banned_users"])
CHANNELS = db["channels"]
save_db()


def get_active_caption_item() -> dict:
    captions = db.get("captions", [])
    if not captions:
        return {}
    idx = db.get("active_caption_index", 0)
    if not (0 <= idx < len(captions)):
        idx = 0
    return captions[idx]


def get_active_caption_text() -> str:
    item = get_active_caption_item()
    return item.get("text", "")


def build_caption_keyboard(caption_item: dict):
    if not caption_item or not caption_item.get("buttons"):
        return None
    builder = InlineKeyboardBuilder()
    for btn in caption_item["buttons"]:
        if isinstance(btn, dict) and btn.get("text") and btn.get("url"):
            builder.button(text=btn["text"], url=btn["url"])
    builder.adjust(1)
    return builder.as_markup()


def save_users():
    db["approved_users"] = list(approved_users)
    db["all_users"] = list(all_users)
    save_db()


def register_user(user_id: int):
    if user_id not in all_users:
        all_users.add(user_id)
        save_users()


# --- 3. КЛАВИАТУРА ПОДПИСКИ ---
def get_sub_keyboard():
    builder = InlineKeyboardBuilder()
    for ch in CHANNELS:
        builder.button(text=ch["name"], url=ch["url"])
    builder.adjust(1)
    builder.row(InlineKeyboardButton(text="✅ Продолжить", callback_data="check_sub"))
    return builder.as_markup()


def get_new_channel_sub_keyboard(channel: dict):
    builder = InlineKeyboardBuilder()
    builder.button(text=channel["name"], url=channel["url"])
    builder.adjust(1)
    builder.row(InlineKeyboardButton(text="✅ Продолжить", callback_data="check_sub"))
    return builder.as_markup()


# --- 3б. ГЛАВНОЕ МЕНЮ ---
def get_main_keyboard():
    return ReplyKeyboardMarkup(
        keyboard=[[KeyboardButton(text="🎬 Скачать Видео 🎬")]],
        resize_keyboard=True,
        is_persistent=True
    )


# --- 4. КОМАНДЫ ПОЛЬЗОВАТЕЛЯ ---
@router.message(Command("start"))
async def cmd_start(message: types.Message, state: FSMContext):
    await state.clear()
    user_id = message.from_user.id
    if user_id in banned_users:
        await message.answer("🚫 Вы заблокированы и не можете пользоваться этим ботом.")
        return

    register_user(user_id)

    # Реклама при старте отправляется ТОЛЬКО если она активирована переключателем в админке
    if db.get("send_as_first_message", False):
        active_cap = get_active_caption_item()
        if active_cap and active_cap.get("text"):
            reply_markup = build_caption_keyboard(active_cap)
            await message.answer(
                text=active_cap["text"],
                parse_mode=ParseMode.HTML,
                disable_web_page_preview=False,
                reply_markup=reply_markup
            )

    await message.answer(
        "💫 Умею скачивать видео из Тик Тока, Инстаграма, YouTube, Пинтереста без водяного знака.\n\n"
        "❤️ ИНСТРУКЦИЯ:\n"
        "1. Скопируй ссылку на видео\n"
        "2. Отправь ссылку в бота\n"
        "3. Получай готовое видео без водяного знака",
        reply_markup=get_main_keyboard(),
        disable_web_page_preview=True
    )


# --- 5. СКАЧИВАНИЕ МЕДИА ---

def get_file_type(path: str) -> str:
    ext = os.path.splitext(path)[1].lower()
    if ext in VIDEO_EXTENSIONS:
        return 'video'
    if ext in PHOTO_EXTENSIONS:
        return 'photo'
    return 'unknown'


def cleanup_files(file_paths: list):
    for path in file_paths:
        try:
            if os.path.exists(path):
                os.remove(path)
        except Exception as e:
            logger.warning(f"Не удалось удалить файл {path}: {e}")


def download_media_files(url: str) -> list[str]:
    uid = str(int(time.time() * 1000))
    outtmpl = f'{DOWNLOAD_DIR}/{uid}_%(autonumber)s.%(ext)s'

    cookiefile = COOKIES_FILE if os.path.exists(COOKIES_FILE) else None

    ydl_opts = {
        'outtmpl': outtmpl,
        'noplaylist': True,
        'quiet': True,
        'no_warnings': True,
        'http_headers': {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'},
        'format': 'best[filesize<50M]/best',
        'write_all_thumbnails': False,
        'postprocessors': [],
        **({'cookiefile': cookiefile} if cookiefile else {}),
    }

    downloaded = []

    with yt_dlp.YoutubeDL(ydl_opts) as ydl:
        info = ydl.extract_info(url, download=True)

        if info.get('_type') == 'playlist' or 'entries' in info:
            entries = info.get('entries') or []
            for entry in entries:
                if entry and entry.get('requested_downloads'):
                    for dl in entry['requested_downloads']:
                        path = dl.get('filepath') or dl.get('filename')
                        if path and os.path.exists(path):
                            downloaded.append(path)
        else:
            if info.get('requested_downloads'):
                for dl in info['requested_downloads']:
                    path = dl.get('filepath') or dl.get('filename')
                    if path and os.path.exists(path):
                        downloaded.append(path)

    if not downloaded:
        pattern = f'{DOWNLOAD_DIR}/{uid}_*'
        found = glob.glob(pattern)
        downloaded = [f for f in found if os.path.exists(f)]

    downloaded = [f for f in downloaded if get_file_type(f) != 'unknown']
    downloaded.sort()

    return downloaded


async def send_media_files(message: types.Message, file_paths: list[str], caption: str) -> list:
    if not file_paths:
        raise ValueError("Нет файлов для отправки")

    sent = []

    # Неудаляемая подпись под самим медиафайлом
    fixed_caption = "🚀 Скачано в @sevVideoBot Пользуйтесь и делитесь с друзьями! 🥰"

    # Реклама отправляется перед видео только если она включена (True) переключателем в админке
    if db.get("send_as_first_message", False) and caption:
        active_cap = get_active_caption_item()
        reply_markup = build_caption_keyboard(active_cap)
        first_msg = await message.answer(
            text=caption,
            parse_mode=ParseMode.HTML,
            disable_web_page_preview=False,
            reply_markup=reply_markup
        )
        sent.append(first_msg)

    if len(file_paths) == 1:
        path = file_paths[0]
        ftype = get_file_type(path)
        if ftype == 'video':
            msg = await message.answer_video(
                video=FSInputFile(path),
                caption=fixed_caption,
                parse_mode=ParseMode.HTML
            )
            sent.append(msg)
        elif ftype == 'photo':
            msg = await message.answer_photo(
                photo=FSInputFile(path),
                caption=fixed_caption,
                parse_mode=ParseMode.HTML
            )
            sent.append(msg)
        else:
            msg = await message.answer_document(
                document=FSInputFile(path),
                caption=fixed_caption,
                parse_mode=ParseMode.HTML
            )
            sent.append(msg)
    else:
        chunks = [file_paths[i:i + 10] for i in range(0, len(file_paths), 10)]

        for chunk_idx, chunk in enumerate(chunks):
            media_group = []
            for i, path in enumerate(chunk):
                ftype = get_file_type(path)
                item_caption = fixed_caption if (chunk_idx == 0 and i == 0) else None

                if ftype == 'video':
                    media_group.append(InputMediaVideo(
                        media=FSInputFile(path),
                        caption=item_caption,
                        parse_mode=ParseMode.HTML if item_caption else None
                    ))
                else:
                    media_group.append(InputMediaPhoto(
                        media=FSInputFile(path),
                        caption=item_caption,
                        parse_mode=ParseMode.HTML if item_caption else None
                    ))

            msgs = await message.answer_media_group(media=media_group)
            if msgs:
                sent.extend(msgs)

    return sent


# --- 6. ОБРАБОТЧИК ССЫЛОК ---
@router.message(F.text.in_({"🎬 Скачать Видео 🎬"}), StateFilter(None))
async def handle_download_button(message: types.Message, state: FSMContext):
    await cmd_start(message, state)


@router.message(F.text, StateFilter(None), ~F.text.startswith("/"))
async def handle_message(message: types.Message):
    user_id = message.from_user.id
    if user_id in banned_users:
        await message.answer("🚫 Вы заблокированы и не можете пользоваться этим ботом.")
        return

    register_user(user_id)
    text = message.text or ""

    url = None
    if message.entities:
        for entity in message.entities:
            if entity.type == "text_link":
                url = entity.url
                break
            elif entity.type == "url":
                url = text[entity.offset: entity.offset + entity.length]
                break

    if not url:
        match = URL_PATTERN.search(text)
        if match:
            url = match.group(0)

    if not url:
        await message.answer("⚠️ Я понимаю только ссылки на видео. Отправьте правильную ссылку!")
        return

    if user_id not in approved_users:
        await message.answer(
            "Чтобы скачивать медиа, подпишись на каналы по кнопкам ниже\n\nПосле нажми «Продолжить»! ✅",
            reply_markup=get_sub_keyboard()
        )
        return

    platform_hint = ""
    lower_url = url.lower()
    if "instagram.com" in lower_url:
        platform_hint = "Instagram"
    elif "youtube.com" in lower_url or "youtu.be" in lower_url:
        platform_hint = "YouTube"
    elif "pinterest.com" in lower_url or "pin.it" in lower_url:
        platform_hint = "Pinterest"

    hint_str = f" с {platform_hint}" if platform_hint else ""

    await bot.send_chat_action(chat_id=message.chat.id, action="upload_video")
    status_msg = await message.answer(f"⏳ <i>Скачиваю медиа{hint_str}...</i>")

    file_paths = []
    try:
        file_paths = await asyncio.to_thread(download_media_files, url)

        if not file_paths:
            await status_msg.edit_text(
                "❌ <b>Не удалось скачать медиа.</b>\n\n"
                "Возможные причины:\n"
                "• Приватный аккаунт или закрытый контент\n"
                "• Видео слишком большое (лимит 50 МБ)\n"
                "• Ссылка недействительна\n\n"
                "Если ссылка точно верная — отправьте её ещё раз! Загрузка иногда происходит со 2-3 раза. 🔄"
            )
            return

        caption = get_active_caption_text()
        sent_messages = await send_media_files(message, file_paths, caption)
        await status_msg.delete()

        if sent_messages:
            media_cache = []
            for msg in sent_messages:
                if msg.video:
                    media_cache.append({"file_id": msg.video.file_id, "type": "video"})
                elif msg.photo:
                    media_cache.append({"file_id": msg.photo[-1].file_id, "type": "photo"})
            if media_cache:
                user_last_media[user_id] = media_cache

    except yt_dlp.utils.DownloadError as e:
        err_str = str(e).lower()
        if "private" in err_str or "login" in err_str or "age" in err_str:
            await status_msg.edit_text(
                "❌ <b>Не удалось скачать.</b>\n\n"
                "Контент приватный или требует авторизации.\n\n"
                "Если ссылка точно верная — отправьте её ещё раз!"
            )
        elif "size" in err_str or "too large" in err_str:
            await status_msg.edit_text(
                "❌ <b>Видео слишком большое.</b>\n\n"
                "Telegram принимает файлы до 50 МБ.\n\n"
                "Если ссылка точно верная — отправьте её ещё раз!"
            )
        else:
            logger.error(f"DownloadError для {url}: {e}")
            await status_msg.edit_text(
                "❌ <b>Ошибка загрузки. Видео приватное, слишком большое или ссылка недействительна.</b>\n\n"
                "Если ссылка точно верная, отправьте её боту еще раз!\n"
                "Загрузка иногда происходит со 2-3 раза. 🔄"
            )
    except Exception as e:
        logger.error(f"Ошибка при обработке {url}: {e}")
        await status_msg.edit_text("❌ <b>Произошла ошибка при загрузке.</b> Попробуйте еще раз.")
    finally:
        cleanup_files(file_paths)


# --- 7. АДМИНКА ---
def get_admin_keyboard():
    builder = InlineKeyboardBuilder()
    builder.button(text="📝 Описания и Реклама", callback_data="admin_captions_menu")
    builder.button(text="📢 Рассылка", callback_data="admin_broadcast")
    builder.button(text="➕ Добавить канал", callback_data="admin_add_channel")
    builder.button(text="📋 Каналы", callback_data="admin_list_channels")
    builder.adjust(2)
    return builder.as_markup()


def get_admin_panel_text():
    return (
        f"🛠 <b>Панель администратора</b>\n\n"
        f"👥 Всего юзеров: <b>{len(all_users)}</b>\n"
        f"✅ Допущено: <b>{len(approved_users)}</b>\n"
        f"🚫 Забанено: <b>{len(banned_users)}</b>\n"
        f"📣 Каналов: <b>{len(CHANNELS)}</b>\n\n"
        f"• <code>/checkuser ID</code> — проверить статус юзера\n"
        f"• <code>/resetuser ID</code> — сбросить допуск юзера\n"
        f"• <code>/ban ID</code> — забанить юзера\n"
        f"• <code>/unban ID</code> — разбанить юзера"
    )


@router.message(Command("admin"))
async def cmd_admin(message: types.Message, state: FSMContext):
    await state.clear()
    if message.from_user.id == ADMIN_ID:
        await message.answer(get_admin_panel_text(), reply_markup=get_admin_keyboard())
    else:
        await message.answer(f"⛔️ <b>В доступе отказано.</b> Ваш ID: <code>{message.from_user.id}</code>")


@router.message(Command("checkuser"))
async def cmd_check_user(message: types.Message):
    if message.from_user.id != ADMIN_ID: return
    parts = message.text.split()
    if len(parts) < 2 or not parts[1].strip().lstrip("-").isdigit():
        await message.answer("Использование: <code>/checkuser USER_ID</code>")
        return
    target_id = int(parts[1].strip())
    is_approved = target_id in approved_users
    is_banned = target_id in banned_users
    in_cd = target_id in sub_check_times
    elapsed = round(time.time() - sub_check_times[target_id], 1) if in_cd else None

    text = f"🔎 <b>Юзер</b> <code>{target_id}</code>\n\n"
    text += f"Бан: {'🚫 Забанен' if is_banned else '✅ Не забанен'}\n"
    text += f"Статус: {'✅ Допущен навсегда' if is_approved else '❌ Ещё не прошёл'}\n"
    if in_cd and not is_approved:
        text += f"⏳ Нажал «Продолжить» {elapsed} сек назад"
    await message.answer(text)


@router.message(Command("ban"))
async def cmd_ban_user(message: types.Message):
    if message.from_user.id != ADMIN_ID: return
    parts = message.text.split()
    if len(parts) < 2 or not parts[1].strip().lstrip("-").isdigit():
        await message.answer("Использование: <code>/ban USER_ID</code>")
        return
    target_id = int(parts[1].strip())
    if target_id == ADMIN_ID:
        await message.answer("❌ Нельзя забанить самого себя.")
        return
    banned_users.add(target_id)
    db["banned_users"] = list(banned_users)
    save_db()
    await message.answer(f"🚫 Юзер <code>{target_id}</code> забанен.")


@router.message(Command("unban"))
async def cmd_unban_user(message: types.Message):
    if message.from_user.id != ADMIN_ID: return
    parts = message.text.split()
    if len(parts) < 2 or not parts[1].strip().lstrip("-").isdigit():
        await message.answer("Использование: <code>/unban USER_ID</code>")
        return
    target_id = int(parts[1].strip())
    was_banned = target_id in banned_users
    banned_users.discard(target_id)
    db["banned_users"] = list(banned_users)
    save_db()
    if was_banned:
        await message.answer(f"✅ Юзер <code>{target_id}</code> разбанен.")
    else:
        await message.answer(f"ℹ️ Юзер <code>{target_id}</code> и так не был забанен.")


@router.message(Command("resetuser"))
async def cmd_reset_user(message: types.Message):
    if message.from_user.id != ADMIN_ID: return
    parts = message.text.split()
    if len(parts) < 2 or not parts[1].strip().lstrip("-").isdigit():
        await message.answer("Использование: <code>/resetuser USER_ID</code>")
        return
    target_id = int(parts[1].strip())
    was_approved = target_id in approved_users
    approved_users.discard(target_id)
    sub_check_times.pop(target_id, None)
    db["approved_users"] = list(approved_users)
    save_db()
    if was_approved:
        await message.answer(f"♻️ Юзер <code>{target_id}</code> сброшен.")
    else:
        await message.answer(f"ℹ️ Юзер <code>{target_id}</code> и так не был допущен.")


@router.callback_query(F.data == "admin_menu")
async def back_to_admin(callback: types.CallbackQuery, state: FSMContext):
    if callback.from_user.id != ADMIN_ID: return
    await state.clear()
    try:
        await callback.message.edit_text(get_admin_panel_text(), reply_markup=get_admin_keyboard())
    except Exception:
        await callback.message.answer(get_admin_panel_text(), reply_markup=get_admin_keyboard())
    await callback.answer()


# --- 8. УПРАВЛЕНИЕ ОПИСАНИЯМИ (РЕКЛАМА) ---
def get_captions_menu_text():
    captions = db.get("captions", [])
    idx = db.get("active_caption_index", 0)
    if not (0 <= idx < len(captions)):
        idx = 0
    active_name = captions[idx]["name"] if captions else "—"
    active_btns = len(captions[idx].get("buttons", [])) if captions else 0

    is_first = db.get("send_as_first_message", False)
    mode_str = "🟢 <b>ВКЛЮЧЕНА (Отдельным сообщением перед видео)</b>" if is_first else "⚪️ <b>ВЫКЛЮЧЕНА (Только ссылка под файлом)</b>"

    return (
        f"📝 <b>Управление рекламными описаниями</b>\n\n"
        f"Текущий выбранный текст: 🟢 <b>{active_name}</b> (Кнопок: {active_btns})\n"
        f"Статус рекламы в боте: {mode_str}\n\n"
        f"<i>Вы можете нажать на любое описание в списке ниже, чтобы переключить бот на него.</i>"
    )


def get_captions_keyboard():
    builder = InlineKeyboardBuilder()
    captions = db.get("captions", [])
    idx = db.get("active_caption_index", 0)
    if not (0 <= idx < len(captions)):
        idx = 0

    for i, cap in enumerate(captions):
        mark = "🟢 " if i == idx else "⚪️ "
        builder.row(
            InlineKeyboardButton(text=f"{mark}{cap['name']}", callback_data=f"cap_activate_{i}"),
            InlineKeyboardButton(text="❌", callback_data=f"cap_delete_{i}")
        )

    is_first = db.get("send_as_first_message", False)
    toggle_text = "⚪️ Отключить рекламу перед видео" if is_first else "🟢 Включить рекламу перед видео"

    builder.row(InlineKeyboardButton(text=toggle_text, callback_data="admin_toggle_first_msg"))
    builder.row(InlineKeyboardButton(text="➕ Добавить новое описание", callback_data="admin_add_caption"))
    builder.row(InlineKeyboardButton(text="⬅️ В меню панели", callback_data="admin_menu"))
    return builder.as_markup()


@router.callback_query(F.data == "admin_captions_menu")
async def captions_menu(callback: types.CallbackQuery, state: FSMContext):
    if callback.from_user.id != ADMIN_ID: return
    await state.clear()
    await callback.message.edit_text(get_captions_menu_text(), reply_markup=get_captions_keyboard())


@router.callback_query(F.data == "admin_toggle_first_msg")
async def toggle_first_msg(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID: return
    db["send_as_first_message"] = not db.get("send_as_first_message", False)
    save_db()
    await callback.answer("Режим рекламы изменен!")
    await callback.message.edit_text(get_captions_menu_text(), reply_markup=get_captions_keyboard())


@router.callback_query(F.data.startswith("cap_activate_"))
async def activate_caption(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID: return
    idx = int(callback.data.replace("cap_activate_", ""))
    captions = db.get("captions", [])
    if 0 <= idx < len(captions):
        db["active_caption_index"] = idx
        save_db()
        await callback.answer(f"✅ Активировано: {captions[idx]['name']}")
        await callback.message.edit_text(get_captions_menu_text(), reply_markup=get_captions_keyboard())
    else:
        await callback.answer("Описание не найдено", show_alert=True)


@router.callback_query(F.data.startswith("cap_delete_"))
async def delete_caption(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID: return
    idx = int(callback.data.replace("cap_delete_", ""))
    captions = db.get("captions", [])

    if len(captions) <= 1:
        await callback.answer("❌ Должно оставаться хотя бы одно описание.", show_alert=True)
        return

    if not (0 <= idx < len(captions)):
        await callback.answer("Описание не найдено", show_alert=True)
        return

    removed = captions.pop(idx)
    db["captions"] = captions

    active_idx = db.get("active_caption_index", 0)
    if idx == active_idx:
        db["active_caption_index"] = 0
    elif idx < active_idx:
        db["active_caption_index"] = active_idx - 1
    save_db()

    await callback.answer(f"Удалено: {removed['name']}")
    await callback.message.edit_text(get_captions_menu_text(), reply_markup=get_captions_keyboard())


@router.callback_query(F.data == "admin_add_caption")
async def add_caption_req(callback: types.CallbackQuery, state: FSMContext):
    if callback.from_user.id != ADMIN_ID: return
    await state.set_state(AdminStates.waiting_for_new_caption)
    await callback.message.edit_text(
        "➕ <b>Новое описание / Реклама с кнопками</b>\n\n"
        "Отправьте данные через разделитель «|»:\n"
        "<code>Название|Текст рекламы|Кнопка 1==ссылка 1|Кнопка 2==ссылка 2</code>\n\n"
        "• <b>Название</b> — видно только вам в админке.\n"
        "• <b>Текст рекламы</b> — увидят пользователи (можно с HTML).\n"
        "• <b>Кнопки (необязательно)</b> — добавляются в конце через <code>|</code>, текст кнопки и ссылка разделяются через <code>==</code>.\n\n"
        "<b>Пример создания рекламы:</b>\n"
        "<code>Реклама ВПН|Попробуйте наш быстрый ВПН сервис!|🎁 ЗАБРАТЬ БЕСПЛАТНО==https://t.me/link1</code>",
        reply_markup=InlineKeyboardBuilder().button(text="⬅️ Отмена", callback_data="admin_captions_menu").as_markup()
    )


@router.message(AdminStates.waiting_for_new_caption)
async def process_new_caption(message: types.Message, state: FSMContext):
    if message.from_user.id != ADMIN_ID: return

    raw = (message.text or "").strip()
    if "|" not in raw:
        await message.answer(
            "❌ Не вижу разделитель «|».\n\n"
            "Нужно отправить строкой:\n"
            "<code>Название|Текст описания</code>"
        )
        return

    parts = raw.split("|")
    name = parts[0].strip()
    text = parts[1].strip()

    buttons = []
    if len(parts) > 2:
        for btn_raw in parts[2:]:
            btn_raw = btn_raw.strip()
            if "==" in btn_raw:
                b_text, b_url = btn_raw.split("==", 1)
                b_text = b_text.strip()
                b_url = b_url.strip()
                if b_text and b_url:
                    buttons.append({"text": b_text, "url": b_url})

    if not name or not text:
        await message.answer("❌ Название или текст пустые.")
        return

    captions = db.get("captions", [])
    captions.append({"name": name, "text": text, "buttons": buttons})
    db["captions"] = captions
    save_db()
    await state.clear()

    await message.answer(
        f"✅ Реклама «{name}» успешно добавлена с {len(buttons)} кн.!\n\n"
        "Откройте меню «📝 Описания и Реклама», чтобы активировать и включить её.",
        reply_markup=get_admin_keyboard()
    )


# --- 9. РАССЫЛКА ---
@router.callback_query(F.data == "admin_broadcast")
async def broadcast_req(callback: types.CallbackQuery, state: FSMContext):
    if callback.from_user.id != ADMIN_ID: return
    await state.set_state(AdminStates.waiting_for_broadcast)
    await callback.message.edit_text(
        f"📢 <b>Рассылка</b>\n\nБудет отправлено <b>{len(all_users)}</b> юзерам.\n\nОтправьте сообщение для рассылки:",
        reply_markup=InlineKeyboardBuilder().button(text="⬅️ Отмена", callback_data="admin_menu").as_markup()
    )


@router.message(AdminStates.waiting_for_broadcast)
async def process_broadcast(message: types.Message, state: FSMContext):
    if message.from_user.id != ADMIN_ID: return
    await state.clear()

    total = len(all_users)
    sent, failed = 0, 0
    status_msg = await message.answer(f"📤 Начинаю рассылку на <b>{total}</b> юзеров...")

    for user_id in list(all_users):
        try:
            await message.copy_to(chat_id=user_id)
            sent += 1
        except Exception:
            failed += 1
        await asyncio.sleep(0.05)

    await status_msg.edit_text(
        f"✅ <b>Рассылка завершена!</b>\n\n📨 Отправлено: <b>{sent}</b>\n❌ Не доставлено: <b>{failed}</b>",
        reply_markup=get_admin_keyboard()
    )


# --- 10. УПРАВЛЕНИЕ КАНАЛАМИ ---
@router.callback_query(F.data == "admin_list_channels")
async def list_channels(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID: return
    builder = InlineKeyboardBuilder()
    text = "<b>Текущие каналы:</b>\n\n"
    if not CHANNELS:
        text += "Каналов нет."
    else:
        for i, ch in enumerate(CHANNELS):
            if not isinstance(ch, dict) or "name" not in ch or "url" not in ch: continue
            text += f"{i + 1}. <b>{ch['name']}</b>\n{ch['url']}\n\n"
            builder.button(text=f"❌ Удалить {ch['name']}", callback_data=f"del_ch_{i}")
    builder.adjust(1)
    builder.row(InlineKeyboardButton(text="⬅️ В меню", callback_data="admin_menu"))
    await callback.message.edit_text(text, reply_markup=builder.as_markup())


@router.callback_query(F.data.startswith("del_ch_"))
async def delete_channel(callback: types.CallbackQuery):
    if callback.from_user.id != ADMIN_ID: return
    idx = int(callback.data.replace("del_ch_", ""))
    if 0 <= idx < len(CHANNELS):
        removed = CHANNELS.pop(idx)
        db["channels"] = CHANNELS
        save_db()
        await callback.answer(f"Канал «{removed['name']}» удалён")
        await list_channels(callback)


@router.callback_query(F.data == "admin_add_channel")
async def add_channel_req(callback: types.CallbackQuery, state: FSMContext):
    if callback.from_user.id != ADMIN_ID: return
    await state.set_state(AdminStates.waiting_for_new_channel)
    await callback.message.edit_text(
        "➕ <b>Добавление канала</b>\n\n"
        "Отправьте данные строкой:\n"
        "<code>Название|https://t.me/ссылка</code>",
        reply_markup=InlineKeyboardBuilder().button(text="⬅️ Отмена", callback_data="admin_menu").as_markup()
    )


@router.message(AdminStates.waiting_for_new_channel)
async def process_new_channel(message: types.Message, state: FSMContext):
    if message.from_user.id != ADMIN_ID: return

    raw = (message.text or "").strip()
    if "|" not in raw:
        await message.answer("❌ Не вижу разделитель «|».")
        return

    name, url = raw.rsplit("|", 1)
    name, url = name.strip(), url.strip()

    if not name or not url.lower().startswith("http"):
        await message.answer("❌ Некорректные данные канала.")
        return

    new_channel = {"name": name, "url": url}
    CHANNELS.append(new_channel)
    db["channels"] = CHANNELS

    approved_users.clear()
    sub_check_times.clear()
    db["approved_users"] = []
    save_db()
    await state.clear()

    total = len(all_users)
    sent, failed = 0, 0
    status_msg = await message.answer(f"✅ Канал добавлен! Отправляю рассылку {total} юзерам...")

    broadcast_text = f"📣 <b>Новый канал!</b>\n\nЧтобы продолжить пользоваться ботом, подпишись:\n\n👉 <b>{name}</b>"

    for user_id in list(all_users):
        try:
            await bot.send_message(chat_id=user_id, text=broadcast_text,
                                   reply_markup=get_new_channel_sub_keyboard(new_channel))
            sent += 1
        except Exception:
            failed += 1
        await asyncio.sleep(0.05)

    await status_msg.edit_text(
        f"✅ Канал добавлен!\n\n📨 Отправлено: <b>{sent}</b>\n❌ Не доставлено: <b>{failed}</b>",
        reply_markup=get_admin_keyboard()
    )


# --- 11. КНОПКА «ПРОДОЛЖИТЬ» С КД 2 СЕК ---
@router.callback_query(F.data == "check_sub")
async def check_sub_callback(callback: types.CallbackQuery):
    user_id = callback.from_user.id
    now = time.time()

    if user_id in banned_users:
        await callback.answer("🚫 Вы заблокированы.", show_alert=True)
        return

    if user_id in approved_users:
        try:
            await callback.message.delete()
        except Exception:
            pass
        await callback.message.answer("🎉 <b>Доступ активирован!</b> Отправьте ссылку ещё раз.",
                                      reply_markup=get_main_keyboard())
        return

    if user_id not in sub_check_times:
        sub_check_times[user_id] = now
        await callback.answer("⏳ Убедитесь, что вы подали заявку на вступление!", show_alert=True)
        return

    if now - sub_check_times[user_id] < 2:
        await callback.answer("⏳ Убедитесь, что вы подали заявку на вступление!", show_alert=True)
        return

    approved_users.add(user_id)
    db["approved_users"] = list(approved_users)
    save_db()
    del sub_check_times[user_id]

    try:
        await callback.message.delete()
    except Exception:
        pass
    await callback.message.answer("🎉 <b>Доступ активирован!</b> Отправьте ссылку ещё раз.",
                                  reply_markup=get_main_keyboard())


# --- 12. ИНЛАЙН-РЕЖИМ ---
@router.inline_query()
async def inline_handler(inline_query: types.InlineQuery):
    user_id = inline_query.from_user.id
    media_list = user_last_media.get(user_id, [])

    if not media_list:
        await inline_query.answer([], switch_pm_text="Сначала скачай видео через бота 👆", switch_pm_parameter="start",
                                  cache_time=1)
        return

    results = []
    for i, item in enumerate(media_list[:50]):
        if item["type"] == "video":
            results.append(
                InlineQueryResultCachedVideo(id=str(i), video_file_id=item["file_id"], title=f"Видео {i + 1}"))
        elif item["type"] == "photo":
            results.append(
                InlineQueryResultCachedPhoto(id=str(i), photo_file_id=item["file_id"], title=f"Фото {i + 1}"))

    await inline_query.answer(results=results, cache_time=1)


# --- 13. ЗАПУСК БОТА ---
async def main():
    await bot.delete_webhook(drop_pending_updates=False)
    logger.info("🚀 Бот успешно запущен и готов к работе!")
    await dp.start_polling(bot, allowed_updates=["message", "callback_query", "inline_query"])


if __name__ == "__main__":
    asyncio.run(main())
