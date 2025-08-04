
# Qqqqqqqq
Yagagag66
import os, json, time, pickle
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes

BOT_TOKEN = "8175697399:AAGClAMq4VV6Uw7gjdhuS3r20dKrO8beyn8"  # ← توكن البوت هنا
COOKIES_DIR = "cookies"
ACCOUNTS_FILE = "accounts.json"

def create_driver():
    chrome_options = Options()
    chrome_options.add_argument("--headless")
    chrome_options.add_argument("--no-sandbox")
    chrome_options.add_argument("--disable-dev-shm-usage")
    chrome_options.add_argument("--disable-gpu")
    chrome_options.add_argument("--window-size=1920x1080")
    chrome_options.binary_location = "/usr/bin/google-chrome"
    return webdriver.Chrome(service=Service("/usr/bin/chromedriver"), options=chrome_options)

def load_accounts():
    if not os.path.exists(ACCOUNTS_FILE):
        return []
    with open(ACCOUNTS_FILE, "r") as f:
        return json.load(f)

def save_accounts(accounts):
    with open(ACCOUNTS_FILE, "w") as f:
        json.dump(accounts, f)

def save_cookies(driver, username):
    os.makedirs(COOKIES_DIR, exist_ok=True)
    with open(f"{COOKIES_DIR}/{username}.pkl", "wb") as f:
        pickle.dump(driver.get_cookies(), f)

def load_cookies(driver, username):
    path = f"{COOKIES_DIR}/{username}.pkl"
    if os.path.exists(path):
        driver.get("https://www.tiktok.com/")
        with open(path, "rb") as f:
            cookies = pickle.load(f)
        for cookie in cookies:
            driver.add_cookie(cookie)
        driver.refresh()
        return True
    return False

def do_action(driver, target, follow=True):
    driver.get(f"https://www.tiktok.com/@{target}")
    time.sleep(5)
    try:
        if follow:
            btn = driver.find_element("xpath", '//button[contains(text(),"Follow")]')
        else:
            btn = driver.find_element("xpath", '//button[contains(text(),"Following")]')
        btn.click()
        return f"✅ {'تمت المتابعة' if follow else 'تم إلغاء المتابعة'} @{target}"
    except Exception as e:
        return f"❌ خطأ: {str(e)}"

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("🤖 أوامر البوت:\n/add username\n/follow username\n/unfollow username\n/delete username\n/count")

async def add_account(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) != 1:
        await update.message.reply_text("❗ استخدم: /add username")
        return
    username = context.args[0]
    accounts = load_accounts()
    if any(a["username"] == username for a in accounts):
        await update.message.reply_text("⚠️ الحساب موجود مسبقًا.")
        return

    await update.message.reply_text(
        f"✅ الرجاء تسجيل الدخول للحساب {username} يدويًا على السيرفر (VNC أو المتصفح)، ثم أرسل:\n/done {username}"
    )

async def mark_done(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) != 1:
        await update.message.reply_text("❗ استخدم: /done username")
        return
    username = context.args[0]

    driver = create_driver()
    driver.get("https://www.tiktok.com/login")
    time.sleep(15)

    save_cookies(driver, username)
    driver.quit()

    accounts = load_accounts()
    if not any(a["username"] == username for a in accounts):
        accounts.append({"username": username})
        save_accounts(accounts)

    await update.message.reply_text(f"✅ تم حفظ الكوكيز والحساب @{username} بنجاح.")

async def follow_or_unfollow(update: Update, context: ContextTypes.DEFAULT_TYPE, follow=True):
    if len(context.args) != 1:
        await update.message.reply_text("❗ استخدم /follow username أو /unfollow username")
        return
    target = context.args[0].replace("@", "")
    accounts = load_accounts()
    results = []
    for acc in accounts:
        driver = create_driver()
        if not load_cookies(driver, acc["username"]):
            results.append(f"{acc['username']}: ❌ لم يسجل الدخول")
            driver.quit()
            continue
        result = do_action(driver, target, follow=follow)
        results.append(f"{acc['username']}: {result}")
        driver.quit()
    await update.message.reply_text("\n".join(results))

async def delete_account(update: Update, context: ContextTypes.DEFAULT_TYPE):
    if len(context.args) != 1:
        await update.message.reply_text("❗ استخدم: /delete username")
        return
    username = context.args[0]
    accounts = [a for a in load_accounts() if a["username"] != username]
    save_accounts(accounts)
    path = f"{COOKIES_DIR}/{username}.pkl"
    if os.path.exists(path):
        os.remove(path)
    await update.message.reply_text(f"🗑️ تم حذف الحساب: {username}")

async def count_accounts(update: Update, context: ContextTypes.DEFAULT_TYPE):
    count = len(load_accounts())
    await update.message.reply_text(f"📊 عدد الحسابات: {count}")

def main():
    app = ApplicationBuilder().token(BOT_TOKEN).build()
    app.add_handler(CommandHandler("start", start))
    app.add_handler(CommandHandler("add", add_account))
    app.add_handler(CommandHandler("done", mark_done))
    app.add_handler(CommandHandler("follow", lambda u, c: follow_or_unfollow(u, c, follow=True)))
    app.add_handler(CommandHandler("unfollow", lambda u, c: follow_or_unfollow(u, c, follow=False)))
    app.add_handler(CommandHandler("delete", delete_account))
    app.add_handler(CommandHandler("count", count_accounts))
    app.run_polling()

if __name__ == "__main__":
    main()
