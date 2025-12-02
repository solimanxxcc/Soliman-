import os
import json
import asyncio
from aiogram import Bot, Dispatcher, types, F
from aiogram.fsm.storage.memory import MemoryStorage
from aiogram.fsm.context import FSMContext
from aiogram.fsm.state import State, StatesGroup
from aiogram.types import InlineKeyboardMarkup, InlineKeyboardButton
from telethon import TelegramClient
from telethon.sessions import StringSession
from telethon.errors import SessionPasswordNeededError
from telethon.tl.types import InputReportReasonSpam, InputReportReasonViolence, InputReportReasonPornography, InputReportReasonChildAbuse, InputReportReasonCopyright, InputReportReasonFake, InputReportReasonIllegalDrugs, InputReportReasonOther
from telethon.tl.functions.messages import ReportRequest

API_ID = 23073896
API_HASH = "aefef8d0db485aab1451f60a4e562cdc"
BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")

ACCOUNTS_DIR = "accounts"
SETTINGS_DIR = "settings"
os.makedirs(ACCOUNTS_DIR, exist_ok=True)
os.makedirs(SETTINGS_DIR, exist_ok=True)

storage = MemoryStorage()
bot = Bot(token=BOT_TOKEN)
dp = Dispatcher(storage=storage)


class Form(StatesGroup):
    phone = State()
    code_input = State()
    password = State()
    report_link = State()
    report_message_link = State()
    report_external_link = State()
    report_message_input = State()
    report_sleep_input = State()
    report_count_input = State()


REPORT_CATEGORIES = {
    "child_abuse": {
        "name": "👶 محتوى ضد الأطفال",
        "subcategories": {
            "child_exploitation": "استغلال أطفال",
            "child_abuse_content": "محتوى إساءة للأطفال",
            "child_grooming": "تحرش بالأطفال",
        }
    },
    "harassment": {
        "name": "🚫 إساءة ومضايقات",
        "subcategories": {
            "offensive_messages": "رسائل مسيئة",
            "insults": "إهانة أو شتائم",
            "continuous_harassment": "مضايقات مستمرة",
            "direct_threat": "تهديد مباشر أو غير مباشر",
            "bullying": "تنمر",
            "blackmail": "ابتزاز شخصي",
            "doxxing": "نشر معلومات شخصية"
        }
    },
    "inappropriate": {
        "name": "⚠️ محتوى غير لائق",
        "subcategories": {
            "sexual_content": "محتوى جنسي",
            "inappropriate_images": "صور غير لائقة",
            "inappropriate_videos": "فيديوهات مخلة",
            "inappropriate_phrases": "عبارات أو إشارات غير مقبولة",
            "violent_content": "محتوى عنيف أو دموي",
            "hate_speech": "تحريض على الكراهية",
        }
    },
    "impersonation": {
        "name": "👤 انتحال",
        "subcategories": {
            "person_impersonation": "انتحال اسم شخص",
            "channel_impersonation": "انتحال هوية قناة/مجموعة",
            "celebrity_impersonation": "انتحال شخصية عامة",
            "photo_theft": "استخدام صور شخص آخر",
            "fake_organization": "إنشاء حساب مزيف باسم منظمة"
        }
    },
    "scam": {
        "name": "💰 احتيال",
        "subcategories": {
            "fraudulent_links": "روابط احتيالية",
            "scam_messages": "رسائل نصب",
            "account_theft": "محاولات سرقة حسابات",
            "money_request": "طلب أموال بطريقة مشبوهة",
            "phishing": "رسائل تصيّد",
        }
    },
    "spam": {
        "name": "📧 سبام وإزعاج",
        "subcategories": {
            "repeated_messages": "رسائل متكررة",
            "unwanted_ads": "إعلانات غير مرغوب فيها",
            "unwanted_links": "نشر روابط غير مرغوبة",
            "annoying_groups": "مجموعات تزعج المستخدمين",
            "unwanted_invites": "دعوات غير مرغوبة لمجموعات/قنوات"
        }
    },
    "security": {
        "name": "🔒 أمان وتهديد",
        "subcategories": {
            "malware": "نشر برامج ضارة",
            "virus_files": "ملفات تحتوي على فيروسات",
            "security_threats": "تهديدات أمنية",
            "hacking_attempts": "محاولات قرصنة",
            "suspicious_files": "ملفات APK أو EXE مشبوهة"
        }
    },
    "illegal": {
        "name": "⚖️ أنشطة غير قانونية",
        "subcategories": {
            "illegal_trade": "تجارة ممنوعة",
            "dangerous_materials": "مواد خطيرة",
            "illegal_content": "ترويج لمحتوى غير قانوني",
            "hacking_tools": "بيع حسابات/أدوات اختراق",
        }
    },
    "copyright": {
        "name": "©️ حقوق الطبع والنشر",
        "subcategories": {
            "unauthorized_content": "نشر محتوى محمي بدون إذن",
            "reuploaded_content": "إعادة رفع فيديوهات/صور/أعمال فنية",
            "pirated_content": "مشاركات تحتوي على محتوى مقرصن"
        }
    }
}


def get_reason_object(category, subcategory):
    mapping = {
        "spam": InputReportReasonSpam(),
        "inappropriate": InputReportReasonPornography(),
        "security": InputReportReasonViolence(),
        "illegal": InputReportReasonIllegalDrugs(),
        "copyright": InputReportReasonCopyright(),
        "impersonation": InputReportReasonFake(),
        "harassment": InputReportReasonViolence(),
        "scam": InputReportReasonFake(),
        "child_abuse": InputReportReasonChildAbuse(),
    }
    return mapping.get(category, InputReportReasonOther())


def load_json(path):
    if not os.path.exists(path):
        return {}
    try:
        with open(path, "r", encoding="utf-8") as f:
            return json.load(f)
    except:
        return {}


def save_json(path, data):
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)


def load_accounts(user_id):
    path = os.path.join(ACCOUNTS_DIR, f"{user_id}.json")
    return load_json(path)


def save_accounts(user_id, accounts):
    path = os.path.join(ACCOUNTS_DIR, f"{user_id}.json")
    save_json(path, accounts)


def load_settings(user_id):
    path = os.path.join(SETTINGS_DIR, f"{user_id}.json")
    settings = load_json(path)
    return {
        "message": settings.get("message", "تم الإبلاغ عن هذا المحتوى"),
        "sleep": settings.get("sleep", 2),
        "count": settings.get("count", 1)
    }


def save_settings(user_id, settings):
    path = os.path.join(SETTINGS_DIR, f"{user_id}.json")
    save_json(path, settings)


def main_menu_keyboard():
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="📢 إبلاغ", callback_data="menu_report")],
        [InlineKeyboardButton(text="⚙️ إعدادات البلاغ", callback_data="menu_settings")],
        [InlineKeyboardButton(text="👥 إدارة الحسابات", callback_data="menu_accounts")]
    ])
    return keyboard


def settings_menu_keyboard():
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="💬 رسالة البلاغ", callback_data="setting_message")],
        [InlineKeyboardButton(text="⏱️ فارق الوقت (ثانية)", callback_data="setting_sleep")],
        [InlineKeyboardButton(text="🔢 عدد البلاغات", callback_data="setting_count")],
        [InlineKeyboardButton(text="📋 عرض الإعدادات", callback_data="setting_view")],
        [InlineKeyboardButton(text="🔙 العودة للقائمة الرئيسية", callback_data="back_to_main")]
    ])
    return keyboard


def report_main_keyboard():
    buttons = []
    for cat_key, cat_data in REPORT_CATEGORIES.items():
        buttons.append([InlineKeyboardButton(text=cat_data["name"], callback_data=f"report_cat_{cat_key}")])
    buttons.append([InlineKeyboardButton(text="🔙 العودة للقائمة الرئيسية", callback_data="back_to_main")])
    return InlineKeyboardMarkup(inline_keyboard=buttons)


def report_subcategory_keyboard(category):
    buttons = []
    subcats = REPORT_CATEGORIES[category]["subcategories"]
    for subcat_key, subcat_name in subcats.items():
        buttons.append([InlineKeyboardButton(text=subcat_name, callback_data=f"report_sub_{category}|{subcat_key}")])
    buttons.append([InlineKeyboardButton(text="🔙 العودة", callback_data="menu_report")])
    return InlineKeyboardMarkup(inline_keyboard=buttons)


def accounts_menu_keyboard():
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="➕ إضافة حساب", callback_data="account_add")],
        [InlineKeyboardButton(text="❌ حذف حساب", callback_data="account_delete")],
        [InlineKeyboardButton(text="📋 عرض الحسابات", callback_data="account_view")],
        [InlineKeyboardButton(text="🔙 العودة للقائمة الرئيسية", callback_data="back_to_main")]
    ])
    return keyboard


def code_input_keyboard():
    buttons = []
    row = []
    for i in range(1, 10):
        row.append(InlineKeyboardButton(text=str(i), callback_data=f"code_num_{i}"))
        if i % 3 == 0:
            buttons.append(row)
            row = []
    buttons.append([
        InlineKeyboardButton(text="0", callback_data="code_num_0"),
        InlineKeyboardButton(text="⌫ مسح", callback_data="code_clear"),
        InlineKeyboardButton(text="✅ إرسال", callback_data="code_submit")
    ])
    buttons.append([InlineKeyboardButton(text="🔙 إلغاء", callback_data="cancel_operation")])
    return InlineKeyboardMarkup(inline_keyboard=buttons)


def cancel_keyboard():
    keyboard = InlineKeyboardMarkup(inline_keyboard=[
        [InlineKeyboardButton(text="🔙 إلغاء", callback_data="cancel_operation")]
    ])
    return keyboard


@dp.message(F.text == "/start")
async def cmd_start(message: types.Message):
    await message.answer(
        "🤖 مرحباً بك في بوت الإبلاغ وإدارة الحسابات\n\n"
        "اختر أحد الخيارات التالية:",
        reply_markup=main_menu_keyboard()
    )


@dp.callback_query(F.data == "back_to_main")
async def back_to_main(callback_query: types.CallbackQuery):
    await callback_query.message.edit_text(
        "🤖 مرحباً بك في بوت الإبلاغ وإدارة الحسابات\n\n"
        "اختر أحد الخيارات التالية:",
        reply_markup=main_menu_keyboard()
    )


@dp.callback_query(F.data == "menu_report")
async def menu_report(callback_query: types.CallbackQuery):
    await callback_query.message.edit_text(
        "📢 نظام الإبلاغ الشامل\n\n"
        "اختر نوع البلاغ المناسب:",
        reply_markup=report_main_keyboard()
    )


@dp.callback_query(F.data == "menu_settings")
async def menu_settings(callback_query: types.CallbackQuery):
    user_id = str(callback_query.from_user.id)
    settings = load_settings(user_id)
    
    await callback_query.message.edit_text(
        "⚙️ إعدادات البلاغ\n\n"
        "اختر الإعداد المطلوب تعديله:",
        reply_markup=settings_menu_keyboard()
    )


@dp.callback_query(F.data == "menu_accounts")
async def menu_accounts(callback_query: types.CallbackQuery):
    await callback_query.message.edit_text(
        "👥 إدارة الحسابات\n\n"
        "اختر العملية المطلوبة:",
        reply_markup=accounts_menu_keyboard()
    )


@dp.callback_query(F.data.startswith("report_cat_"))
async def report_category_selected(callback_query: types.CallbackQuery):
    category = callback_query.data.replace("report_cat_", "")
    cat_name = REPORT_CATEGORIES[category]["name"]
    await callback_query.message.edit_text(
        f"{cat_name}\n\n"
        "اختر التصنيف الفرعي المناسب:",
        reply_markup=report_subcategory_keyboard(category)
    )


@dp.callback_query(F.data.startswith("report_sub_"))
async def report_subcategory_selected(callback_query: types.CallbackQuery, state: FSMContext):
    parts = callback_query.data.replace("report_sub_", "").split("|")
    category = parts[0]
    subcategory = parts[1]
    
    await state.update_data(report_category=category, report_subcategory=subcategory)
    
    subcat_name = REPORT_CATEGORIES[category]["subcategories"][subcategory]
    cat_name = REPORT_CATEGORIES[category]["name"]
    await callback_query.message.edit_text(
        f"📍 النوع: {cat_name}\n"
        f"📝 التصنيف: {subcat_name}\n\n"
        "الرجاء إرسال روابط الرسائل/القنوات المراد الإبلاغ عنها (رابط واحد أو أكثر، كل رابط على سطر جديد):",
        reply_markup=cancel_keyboard()
    )
    await state.set_state(Form.report_link)


@dp.message(Form.report_link)
async def receive_report_link(message: types.Message, state: FSMContext):
    links = [link.strip() for link in message.text.strip().split('\n') if link.strip()]
    
    user_id = str(message.from_user.id)
    accounts = load_accounts(user_id)
    settings = load_settings(user_id)
    
    if not accounts:
        await message.answer(
            "❌ ليس لديك حسابات مسجلة.\n"
            "يرجى إضافة حساب أولاً من قائمة إدارة الحسابات.",
            reply_markup=main_menu_keyboard()
        )
        await state.clear()
        return
    
    data = await state.get_data()
    category = data.get("report_category")
    subcategory = data.get("report_subcategory")
    
    category_name = REPORT_CATEGORIES[category]["name"]
    subcategory_name = REPORT_CATEGORIES[category]["subcategories"][subcategory]
    
    total_reports = len(links) * settings["count"] * len(accounts)
    
    status_msg = await message.answer(
        f"📊 لوحة حالة البلاغ\n\n"
        f"📝 نوع البلاغ: {category_name}\n"
        f"🏷️ التصنيف: {subcategory_name}\n"
        f"💬 الرسالة: \"{settings['message']}\"\n"
        f"⏱️ السليب: {settings['sleep']} ثانية\n"
        f"🔢 عدد البلاغات المطلوب: {settings['count']}\n"
        f"🔗 عدد الروابط: {len(links)}\n"
        f"👥 عدد الحسابات: {len(accounts)}\n"
        f"📊 إجمالي البلاغات: {total_reports}\n\n"
        f"🔄 رقم البلاغ الحالي: 0/{total_reports}\n\n"
        f"⏳ جاري معالجة البلاغات..."
    )
    
    report_count = 0
    for link_idx, link in enumerate(links):
        for account_idx, (phone, account_data) in enumerate(accounts.items()):
            for report_num in range(1, settings["count"] + 1):
                try:
                    session_str = account_data.get("session")
                    client = TelegramClient(StringSession(session_str), API_ID, API_HASH)
                    await client.connect()
                    
                    reason = get_reason_object(category, subcategory)
                    await client(ReportRequest(link, reason=reason))
                    
                    report_count += 1
                    
                    await status_msg.edit_text(
                        f"📊 لوحة حالة البلاغ\n\n"
                        f"📝 نوع البلاغ: {category_name}\n"
                        f"🏷️ التصنيف: {subcategory_name}\n"
                        f"💬 الرسالة: \"{settings['message']}\"\n"
                        f"⏱️ السليب: {settings['sleep']} ثانية\n"
                        f"🔢 عدد البلاغات المطلوب: {settings['count']}\n"
                        f"🔗 عدد الروابط: {len(links)}\n"
                        f"👥 عدد الحسابات: {len(accounts)}\n"
                        f"📊 إجمالي البلاغات: {total_reports}\n\n"
                        f"🔄 رقم البلاغ الحالي: {report_count}/{total_reports}\n\n"
                        f"⏳ جاري معالجة البلاغات..."
                    )
                    
                    await client.disconnect()
                    
                    if report_count < total_reports:
                        await asyncio.sleep(settings["sleep"])
                    
                except Exception as e:
                    pass
    
    await status_msg.edit_text(
        f"✅ تمت معالجة جميع البلاغات\n\n"
        f"📝 نوع البلاغ: {category_name}\n"
        f"🏷️ التصنيف: {subcategory_name}\n"
        f"💬 الرسالة: \"{settings['message']}\"\n"
        f"⏱️ السليب: {settings['sleep']} ثانية\n"
        f"🔢 عدد البلاغات المطلوب: {settings['count']}\n"
        f"🔗 عدد الروابط: {len(links)}\n"
        f"👥 عدد الحسابات: {len(accounts)}\n"
        f"📊 إجمالي البلاغات المرسلة: {report_count}",
        reply_markup=main_menu_keyboard()
    )
    
    await state.clear()


@dp.callback_query(F.data == "cancel_operation")
async def cancel_operation(callback_query: types.CallbackQuery, state: FSMContext):
    await state.clear()
    await callback_query.message.edit_text(
        "🤖 مرحباً بك في بوت الإبلاغ وإدارة الحسابات\n\n"
        "اختر أحد الخيارات التالية:",
        reply_markup=main_menu_keyboard()
    )


@dp.callback_query(F.data == "account_add")
async def account_add(callback_query: types.CallbackQuery, state: FSMContext):
    await callback_query.message.edit_text(
        "📱 الرجاء إدخال رقم الهاتف (بصيغة دولية مثل: +201234567890):",
        reply_markup=cancel_keyboard()
    )
    await state.set_state(Form.phone)


@dp.message(Form.phone)
async def receive_phone(message: types.Message, state: FSMContext):
    phone = message.text.strip()
    await state.update_data(phone=phone)
    
    user_id = str(message.from_user.id)
    accounts = load_accounts(user_id)
    
    if phone in accounts:
        await message.answer(
            "⚠️ هذا الحساب مسجل بالفعل.",
            reply_markup=accounts_menu_keyboard()
        )
        await state.clear()
        return
    
    try:
        client = TelegramClient(StringSession(), API_ID, API_HASH)
        await client.connect()
        
        await client.send_code_request(phone)
        
        await state.update_data(client=client)
        await message.answer(
            "✅ تم إرسال الكود.\n\n"
            "الرجاء إدخال الكود (استخدم الأزرار أو اكتب الرقم مباشرة):",
            reply_markup=code_input_keyboard()
        )
        await state.set_state(Form.code_input)
    except Exception as e:
        await message.answer(
            f"❌ خطأ: {str(e)}\n\n"
            "الرجاء المحاولة مرة أخرى.",
            reply_markup=accounts_menu_keyboard()
        )
        await state.clear()


@dp.callback_query(F.data.startswith("code_num_"))
async def code_button_pressed(callback_query: types.CallbackQuery, state: FSMContext):
    digit = callback_query.data.replace("code_num_", "")
    data = await state.get_data()
    current_code = data.get("code", "")
    new_code = current_code + digit
    
    await state.update_data(code=new_code)
    await callback_query.message.edit_text(
        f"الكود المدخل: {'*' * len(new_code)}\n\n"
        "الرجاء إدخال الكود:",
        reply_markup=code_input_keyboard()
    )


@dp.callback_query(F.data == "code_clear")
async def code_clear(callback_query: types.CallbackQuery, state: FSMContext):
    await state.update_data(code="")
    await callback_query.message.edit_text(
        "تم مسح الكود\n\n"
        "الرجاء إدخال الكود:",
        reply_markup=code_input_keyboard()
    )


@dp.callback_query(F.data == "code_submit")
async def code_submit(callback_query: types.CallbackQuery, state: FSMContext):
    data = await state.get_data()
    code = data.get("code", "")
    
    if not code:
        await callback_query.answer("❌ الرجاء إدخال الكود أولاً", show_alert=True)
        return
    
    try:
        phone = data.get("phone")
        client = data.get("client")
        
try:
    await client.sign_in(phone=phone, code=code)

except SessionPasswordNeededError:
    await state.set_state(Form.password)
    await message.answer("🔐 الحساب محمي بكلمة مرور. الرجاء إدخال كلمة المرور.")
    return

except Exception as e:
    if "CODE_INVALID" in str(e) or "phone_code_invalid" in str(e).lower():
        await message.answer("❌ كود التحقق غير صحيح. حاول مرة أخرى.")
        return
    else:
        await message.answer(f"⚠️ خطأ غير متوقع: {e}")
        return

# إذا وصل البرنامج هنا فهذا يعني أن تسجيل الدخول نجح
user_id = str(callback_query.from_user.id)
accounts = load_accounts(user_id)
accounts[phone] = {"session": client.session.save()}
save_accounts(user_id, accounts)

await callback_query.message.edit_text(
    f"✅ تم إضافة الحساب: {phone}\n\n"
    "يمكنك الآن الاستخدام."
)
            reply_markup=accounts_menu_keyboard()
        )
        await state.clear()
    except Exception as e:
        await callback_query.message.edit_text(
            f"❌ خطأ: {str(e)}",
            reply_markup=accounts_menu_keyboard()
        )
        await state.clear()


@dp.callback_query(F.data == "account_delete")
async def account_delete(callback_query: types.CallbackQuery):
    user_id = str(callback_query.from_user.id)
    accounts = load_accounts(user_id)
    
    if not accounts:
        await callback_query.message.edit_text(
            "❌ ليس لديك حسابات مسجلة.",
            reply_markup=accounts_menu_keyboard()
        )
        return
    
    buttons = []
    for phone in accounts.keys():
        buttons.append([InlineKeyboardButton(text=f"❌ {phone}", callback_data=f"delete_account_{phone}")])
    buttons.append([InlineKeyboardButton(text="🔙 رجوع", callback_data="back_to_accounts")])
    
    keyboard = InlineKeyboardMarkup(inline_keyboard=buttons)
    await callback_query.message.edit_text(
        "اختر الحساب المراد حذفه:",
        reply_markup=keyboard
    )


@dp.callback_query(F.data.startswith("delete_account_"))
async def delete_account_confirm(callback_query: types.CallbackQuery):
    phone = callback_query.data.replace("delete_account_", "")
    user_id = str(callback_query.from_user.id)
    accounts = load_accounts(user_id)
    
    if phone in accounts:
        del accounts[phone]
        save_accounts(user_id, accounts)
        
        await callback_query.message.edit_text(
            f"✅ تم حذف الحساب: {phone}",
            reply_markup=accounts_menu_keyboard()
        )
    else:
        await callback_query.message.edit_text(
            "❌ لم يتم العثور على الحساب.",
            reply_markup=accounts_menu_keyboard()
        )


@dp.callback_query(F.data == "account_view")
async def account_view(callback_query: types.CallbackQuery):
    user_id = str(callback_query.from_user.id)
    accounts = load_accounts(user_id)
    
    if not accounts:
        await callback_query.message.edit_text(
            "❌ ليس لديك حسابات مسجلة.",
            reply_markup=accounts_menu_keyboard()
        )
        return
    
    accounts_list = "\n".join([f"📱 {phone}" for phone in accounts.keys()])
    await callback_query.message.edit_text(
        f"📋 حساباتك المسجلة:\n\n{accounts_list}",
        reply_markup=accounts_menu_keyboard()
    )


@dp.callback_query(F.data == "back_to_accounts")
async def back_to_accounts(callback_query: types.CallbackQuery):
    await callback_query.message.edit_text(
        "👥 إدارة الحسابات\n\n"
        "اختر العملية المطلوبة:",
        reply_markup=accounts_menu_keyboard()
    )


@dp.callback_query(F.data == "setting_message")
async def setting_message(callback_query: types.CallbackQuery, state: FSMContext):
    user_id = str(callback_query.from_user.id)
    settings = load_settings(user_id)
    
    await callback_query.message.edit_text(
        f"💬 رسالة البلاغ الحالية:\n\n\"{settings['message']}\"\n\n"
        "الرجاء إدخال الرسالة الجديدة:",
        reply_markup=cancel_keyboard()
    )
    await state.set_state(Form.report_message_input)


@dp.message(Form.report_message_input)
async def receive_report_message(message: types.Message, state: FSMContext):
    user_id = str(message.from_user.id)
    new_message = message.text.strip()
    
    settings = load_settings(user_id)
    settings["message"] = new_message
    save_settings(user_id, settings)
    
    await message.answer(
        f"✅ تم تحديث الرسالة:\n\n\"{new_message}\"",
        reply_markup=settings_menu_keyboard()
    )
    await state.clear()


@dp.callback_query(F.data == "setting_sleep")
async def setting_sleep(callback_query: types.CallbackQuery, state: FSMContext):
    user_id = str(callback_query.from_user.id)
    settings = load_settings(user_id)
    
    await callback_query.message.edit_text(
        f"⏱️ فارق الوقت الحالي: {settings['sleep']} ثانية\n\n"
        "الرجاء إدخال فارق الوقت الجديد (بالثواني):",
        reply_markup=cancel_keyboard()
    )
    await state.set_state(Form.report_sleep_input)


@dp.message(Form.report_sleep_input)
async def receive_report_sleep(message: types.Message, state: FSMContext):
    user_id = str(message.from_user.id)
    
    try:
        sleep_time = int(message.text.strip())
        if sleep_time < 0:
            raise ValueError("يجب أن يكون الرقم موجب")
        
        settings = load_settings(user_id)
        settings["sleep"] = sleep_time
        save_settings(user_id, settings)
        
        await message.answer(
            f"✅ تم تحديث فارق الوقت: {sleep_time} ثانية",
            reply_markup=settings_menu_keyboard()
        )
    except ValueError:
        await message.answer(
            "❌ الرجاء إدخال رقم صحيح موجب",
            reply_markup=cancel_keyboard()
        )
        return
    
    await state.clear()


@dp.callback_query(F.data == "setting_count")
async def setting_count(callback_query: types.CallbackQuery, state: FSMContext):
    user_id = str(callback_query.from_user.id)
    settings = load_settings(user_id)
    
    await callback_query.message.edit_text(
        f"🔢 عدد البلاغات الحالي: {settings['count']}\n\n"
        "الرجاء إدخال عدد البلاغات الجديد:",
        reply_markup=cancel_keyboard()
    )
    await state.set_state(Form.report_count_input)


@dp.message(Form.report_count_input)
async def receive_report_count(message: types.Message, state: FSMContext):
    user_id = str(message.from_user.id)
    
    try:
        count = int(message.text.strip())
        if count <= 0:
            raise ValueError("يجب أن يكون الرقم أكبر من صفر")
        
        settings = load_settings(user_id)
        settings["count"] = count
        save_settings(user_id, settings)
        
        await message.answer(
            f"✅ تم تحديث عدد البلاغات: {count}",
            reply_markup=settings_menu_keyboard()
        )
    except ValueError:
        await message.answer(
            "❌ الرجاء إدخال رقم صحيح موجب",
            reply_markup=cancel_keyboard()
        )
        return
    
    await state.clear()


@dp.callback_query(F.data == "setting_view")
async def setting_view(callback_query: types.CallbackQuery):
    user_id = str(callback_query.from_user.id)
    settings = load_settings(user_id)
    
    await callback_query.message.edit_text(
        f"📋 إعدادات البلاغ الحالية:\n\n"
        f"💬 الرسالة: \"{settings['message']}\"\n"
        f"⏱️ فارق الوقت: {settings['sleep']} ثانية\n"
        f"🔢 عدد البلاغات: {settings['count']}",
        reply_markup=settings_menu_keyboard()
    )


async def main():
    await dp.start_polling(bot)


if __name__ == "__main__":
    asyncio.run(main())