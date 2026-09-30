import sqlite3
import time
from collections import defaultdict
from telegram import Update, ChatPermissions
from telegram.ext import Application, CommandHandler, MessageHandler, ChatMemberHandler, ContextTypes, filters

BOT_TOKEN = "8368224966:AAGGoohgxTXJune2s3Sj1tYbI-LuqHkFY7U"
DB_NAME = "rose_style_bot.db"
SPAM_LIMIT = 6
SPAM_WINDOW = 10

db = sqlite3.connect(DB_NAME, check_same_thread=False)
cursor = db.cursor()
cursor.execute("""CREATE TABLE IF NOT EXISTS warnings (chat_id INTEGER, user_id INTEGER, warns INTEGER DEFAULT 0, PRIMARY KEY(chat_id,user_id))""")
cursor.execute("""CREATE TABLE IF NOT EXISTS filters (chat_id INTEGER, keyword TEXT, response TEXT, PRIMARY KEY(chat_id,keyword))""")
cursor.execute("""CREATE TABLE IF NOT EXISTS settings (chat_id INTEGER PRIMARY KEY, welcome TEXT DEFAULT 'Welcome {name} to {chat}!')""")
db.commit()
spam_tracker = defaultdict(list)

async def is_admin(update, user_id=None):
    if not update.effective_chat: return False
    user_id = user_id or update.effective_user.id
    try:
        m = await update.effective_chat.get_member(user_id)
        return m.status in ("administrator","creator")
    except Exception: return False

async def admin_required(update):
    if not await is_admin(update):
        await update.message.reply_text("❌ This command is only for group admins.")
        return False
    return True

def target(update):
    return update.message.reply_to_message.from_user if update.message and update.message.reply_to_message else None

async def start(update, context):
    await update.message.reply_text("🤖 MyRoseBot is online!\n\nUse /help to see commands.")

async def help_command(update, context):
    await update.message.reply_text("""🤖 MyRoseBot

Moderation:
/ban /unban /kick /mute /tmute /unmute
/warn /warns /unwarn

Admin:
/admins /id /ping

Welcome:
/setwelcome <message>
/welcome

Filters:
/filter <word> <reply>
/filters
/stop <word>

Reply to a user's message for moderation commands.""")

async def id_command(update, context):
    u = target(update)
    if u:
        await update.message.reply_text(f"User: {u.full_name}\nID: {u.id}")
    else:
        await update.message.reply_text(f"Your ID: {update.effective_user.id}\nChat ID: {update.effective_chat.id}")

async def ping_command(update, context):
    await update.message.reply_text("🏓 Pong! Bot is online.")

async def admins_command(update, context):
    admins = await update.effective_chat.get_administrators()
    await update.message.reply_text("👮 Admins\n\n" + "\n".join(f"• {a.user.full_name} — {a.user.id}" for a in admins))

async def ban(update, context):
    if not await admin_required(update): return
    u = target(update)
    if not u: return await update.message.reply_text("❌ Reply to a user's message and use /ban.")
    if await is_admin(update,u.id): return await update.message.reply_text("❌ You cannot ban an admin.")
    try:
        await update.effective_chat.ban_member(u.id)
        await update.message.reply_text(f"🔨 {u.full_name} has been banned.")
    except Exception as e: await update.message.reply_text(f"❌ Ban failed: {e}")

async def unban(update, context):
    if not await admin_required(update): return
    if not context.args: return await update.message.reply_text("Usage: /unban USER_ID")
    try:
        uid=int(context.args[0]); await update.effective_chat.unban_member(uid, only_if_banned=True)
        await update.message.reply_text(f"✅ User {uid} has been unbanned.")
    except Exception as e: await update.message.reply_text(f"❌ Unban failed: {e}")

async def kick(update, context):
    if not await admin_required(update): return
    u=target(update)
    if not u: return await update.message.reply_text("❌ Reply to the user's message and use /kick.")
    if await is_admin(update,u.id): return await update.message.reply_text("❌ You cannot kick an admin.")
    try:
        await update.effective_chat.ban_member(u.id); await update.effective_chat.unban_member(u.id)
        await update.message.reply_text(f"👢 {u.full_name} has been kicked.")
    except Exception as e: await update.message.reply_text(f"❌ Kick failed: {e}")

async def mute(update, context):
    if not await admin_required(update): return
    u=target(update)
    if not u: return await update.message.reply_text("❌ Reply to a user's message and use /mute.")
    if await is_admin(update,u.id): return await update.message.reply_text("❌ You cannot mute an admin.")
    try:
        await update.get_bot().restrict_chat_member(update.effective_chat.id,u.id,permissions=ChatPermissions(can_send_messages=False))
        await update.message.reply_text(f"🔇 {u.full_name} has been muted.")
    except Exception as e: await update.message.reply_text(f"❌ Mute failed: {e}")

async def tmute(update, context):
    if not await admin_required(update): return
    u=target(update)
    if not u or not context.args: return await update.message.reply_text("Usage: reply to user + /tmute MINUTES")
    try:
        minutes=int(context.args[0])
        until=int(time.time())+minutes*60
        await update.get_bot().restrict_chat_member(update.effective_chat.id,u.id,permissions=ChatPermissions(can_send_messages=False),until_date=until)
        await update.message.reply_text(f"🔇 {u.full_name} muted for {minutes} minutes.")
    except Exception as e: await update.message.reply_text(f"❌ Temporary mute failed: {e}")

async def unmute(update, context):
    if not await admin_required(update): return
    u=target(update)
    if not u: return await update.message.reply_text("❌ Reply to the muted user's message.")
    try:
        p=ChatPermissions(can_send_messages=True,can_send_audios=True,can_send_documents=True,can_send_photos=True,can_send_videos=True,can_send_video_notes=True,can_send_voice_notes=True,can_send_polls=True,can_send_other_messages=True,can_add_web_page_previews=True)
        await update.get_bot().restrict_chat_member(update.effective_chat.id,u.id,permissions=p)
        await update.message.reply_text(f"🔊 {u.full_name} has been unmuted.")
    except Exception as e: await update.message.reply_text(f"❌ Unmute failed: {e}")

async def warn(update, context):
    if not await admin_required(update): return
    u=target(update)
    if not u: return await update.message.reply_text("❌ Reply to a user's message and use /warn.")
    if await is_admin(update,u.id): return await update.message.reply_text("❌ You cannot warn an admin.")
    cid=update.effective_chat.id
    cursor.execute("SELECT warns FROM warnings WHERE chat_id=? AND user_id=?",(cid,u.id)); row=cursor.fetchone()
    n=row[0]+1 if row else 1
    cursor.execute("INSERT OR REPLACE INTO warnings VALUES(?,?,?)",(cid,u.id,n)); db.commit()
    if n>=3:
        try:
            await update.get_bot().restrict_chat_member(cid,u.id,permissions=ChatPermissions(can_send_messages=False))
            cursor.execute("DELETE FROM warnings WHERE chat_id=? AND user_id=?",(cid,u.id)); db.commit()
            await update.message.reply_text(f"⚠️ {u.full_name} reached 3 warnings.\n🔇 User has been muted.")
        except Exception as e: await update.message.reply_text(f"⚠️ Mute failed: {e}")
    else: await update.message.reply_text(f"⚠️ Warning given to {u.full_name}.\nWarnings: {n}/3")

async def warns(update, context):
    u=target(update)
    if not u: return await update.message.reply_text("❌ Reply to a user and use /warns.")
    cursor.execute("SELECT warns FROM warnings WHERE chat_id=? AND user_id=?",(update.effective_chat.id,u.id)); row=cursor.fetchone()
    await update.message.reply_text(f"⚠️ {u.full_name} has {row[0] if row else 0}/3 warnings.")

async def unwarn(update, context):
    if not await admin_required(update): return
    u=target(update)
    if not u: return await update.message.reply_text("❌ Reply to a user and use /unwarn.")
    cursor.execute("DELETE FROM warnings WHERE chat_id=? AND user_id=?",(update.effective_chat.id,u.id)); db.commit()
    await update.message.reply_text(f"✅ Warnings reset for {u.full_name}.")

async def welcome(update, context):
    cursor.execute("SELECT welcome FROM settings WHERE chat_id=?",(update.effective_chat.id,)); row=cursor.fetchone()
    await update.message.reply_text("👋 Current welcome:\n"+(row[0] if row else "Welcome {name} to {chat}!"))

async def setwelcome(update, context):
    if not await admin_required(update): return
    if not context.args: return await update.message.reply_text("Usage: /setwelcome Welcome {name} to {chat}!")
    msg=" ".join(context.args)
    cursor.execute("INSERT OR REPLACE INTO settings(chat_id,welcome) VALUES(?,?)",(update.effective_chat.id,msg)); db.commit()
    await update.message.reply_text("✅ Welcome message saved.")

async def new_member(update, context):
    r=update.chat_member
    if r and r.old_chat_member.status in ("left","kicked") and r.new_chat_member.status=="member":
        cursor.execute("SELECT welcome FROM settings WHERE chat_id=?",(update.effective_chat.id,)); row=cursor.fetchone()
        msg=row[0] if row else "Welcome {name} to {chat}! 👋"
        msg=msg.replace("{name}",r.new_chat_member.user.full_name).replace("{chat}",update.effective_chat.title or "the group")
        await update.effective_chat.send_message(msg)

async def add_filter(update, context):
    if not await admin_required(update): return
    if len(context.args)<2: return await update.message.reply_text("Usage: /filter word reply")
    word=context.args[0].lower(); reply=" ".join(context.args[1:])
    cursor.execute("INSERT OR REPLACE INTO filters VALUES(?,?,?)",(update.effective_chat.id,word,reply)); db.commit()
    await update.message.reply_text(f"✅ Filter added: {word}")

async def list_filters(update, context):
    cursor.execute("SELECT keyword FROM filters WHERE chat_id=?",(update.effective_chat.id,)); rows=cursor.fetchall()
    await update.message.reply_text("📭 No filters." if not rows else "🔎 Filters:\n\n" + "\n".join("• "+r[0] for r in rows))

async def stop_filter(update, context):
    if not await admin_required(update): return
    if not context.args: return await update.message.reply_text("Usage: /stop word")
    word=context.args[0].lower()
    cursor.execute("DELETE FROM filters WHERE chat_id=? AND keyword=?",(update.effective_chat.id,word)); db.commit()
    await update.message.reply_text(f"✅ Filter removed: {word}")

async def message_handler(update, context):
    if not update.message or update.effective_chat.type not in ("group","supergroup"): return
    u=update.effective_user
    if await is_admin(update,u.id): return
    now=time.time(); key=(update.effective_chat.id,u.id)
    spam_tracker[key]=[t for t in spam_tracker[key] if now-t<SPAM_WINDOW]
    spam_tracker[key].append(now)
    if len(spam_tracker[key])>SPAM_LIMIT:
        try:
            await update.message.delete()
            await update.get_bot().restrict_chat_member(update.effective_chat.id,u.id,permissions=ChatPermissions(can_send_messages=False),until_date=int(time.time())+60)
            await update.effective_chat.send_message(f"🚫 {u.full_name} muted for 60 seconds due to spam.")
            spam_tracker[key]=[]
        except Exception: pass
        return
    text=update.message.text
    if not text: return
    cursor.execute("SELECT keyword,response FROM filters WHERE chat_id=?",(update.effective_chat.id,))
    for word,reply in cursor.fetchall():
        if word in text.lower():
            await update.message.reply_text(reply); break

async def error_handler(update, context):
    print("ERROR:", context.error)

def main():
    if BOT_TOKEN=="8368224966:AAFMj2S9WFCXykGuaoQnyY8y8GRWyYh7UfQ":
        print("ERROR: Put your BotFather token in BOT_TOKEN.")
        return
    app=Application.builder().token(BOT_TOKEN).build()
    handlers=[
        ("start",start),("help",help_command),("id",id_command),("ping",ping_command),
        ("admins",admins_command),("ban",ban),("unban",unban),("kick",kick),
        ("mute",mute),("tmute",tmute),("unmute",unmute),("warn",warn),
        ("warns",warns),("unwarn",unwarn),("welcome",welcome),
        ("setwelcome",setwelcome),("filter",add_filter),("filters",list_filters),
        ("stop",stop_filter)
    ]
    for name, func in handlers:
        app.add_handler(CommandHandler(name,func))
    app.add_handler(ChatMemberHandler(new_member,ChatMemberHandler.CHAT_MEMBER))
    app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND,message_handler))
    app.add_error_handler(error_handler)
    print("🤖 MyBlackberry is starting...")
    app.run_polling()

if __name__=="__main__":
    main()
