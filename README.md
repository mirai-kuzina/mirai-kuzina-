import discord
from discord.ext import commands
import time
import random
import asyncio
import requests
import re
from datetime import datetime, timedelta
from flask import Flask, render_template_string
from threading import Thread

# ==================== MÃ MÀU TERMINAL ====================
class Colors:
    HEADER = '\033[95m'
    BLUE = '\033[94m'
    CYAN = '\033[96m'
    GREEN = '\033[92m'
    WARNING = '\033[93m'
    FAIL = '\033[91m'
    ENDC = '\033[0m'
    BOLD = '\033[1m'

# ==================== VÁ LỖI DISCORD 32-BIT ====================
from discord.state import ConnectionState
from discord.settings import Settings

def patched_parse_ready_supplemental(self, data):
    if not data: return
    try:
        data.pop('user_settings', None)
        data.pop('friend_source_flags', None)
    except: pass
    return old_parse_ready_supplemental(self, data)

old_parse_ready_supplemental = ConnectionState.parse_ready_supplemental
ConnectionState.parse_ready_supplemental = patched_parse_ready_supplemental

old_settings_init = Settings.__init__
def patched_settings_init(self, *, data, state):
    if data is None: data = {}
    if 'friend_source_flags' not in data or data['friend_source_flags'] is None:
        data['friend_source_flags'] = {'all': True, 'mutual_friends': True, 'mutual_guilds': True}
    old_settings_init(self, data=data, state=state)
Settings.__init__ = patched_settings_init

# ==================== CẤU HÌNH HỆ THỐNG ====================
OWNER_NAME = "PRO_MASTER_FARMER"
YOUR_USER_ID = 1060188035741925426  
DISCORD_WEBHOOK_URL = "ĐIỀN_WEBHOOK_MỚI_VÀO_ĐÂY"
TOKEN_TAI_KHOAN_FARM = "ĐIỀN_TOKEN_MỚI_VÀO_ĐÂY"

FARM_CHANNEL_IDS = [1516766378768470067, 1516766332417347658] 
LONG_COOLDOWN_COMMANDS = ["owo piku", "owopup", "oworun"] 

# ==================== BIẾN THỐNG KÊ (DÀNH CHO WEB) ====================
start_time = time.time()
stats = {
    "hunts": 0,
    "battles": 0,
    "captchas": 0,
    "gems_used": {},
    "logs": []
}

def add_log(message):
    """Thêm log vào web (Giữ tối đa 30 dòng gần nhất)"""
    timestamp = datetime.now().strftime('%H:%M:%S')
    stats["logs"].insert(0, f"[{timestamp}] {message}")
    if len(stats["logs"]) > 30:
        stats["logs"].pop()

# ==================== GIAO DIỆN WEB (FLASK) ====================
app = Flask(__name__)

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>OwO Bot Dashboard</title>
    <!-- Tự động làm mới trang mỗi 5 giây -->
    <meta http-equiv="refresh" content="5">
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background-color: #121212; color: #ffffff; margin: 0; padding: 20px; }
        .container { max-width: 900px; margin: 0 auto; }
        .header { text-align: center; border-bottom: 2px solid #00ffcc; padding-bottom: 10px; margin-bottom: 20px; }
        .header h1 { margin: 0; color: #00ffcc; }
        .header p { color: #aaaaaa; font-size: 14px; }
        .stats-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 15px; margin-bottom: 20px; }
        .stat-card { background: #1e1e1e; padding: 20px; border-radius: 8px; text-align: center; border-left: 4px solid #00ffcc; box-shadow: 0 4px 6px rgba(0,0,0,0.3); }
        .stat-card h3 { margin: 0 0 10px 0; color: #888; font-size: 16px; text-transform: uppercase; }
        .stat-card .value { font-size: 28px; font-weight: bold; color: #fff; }
        .danger { border-left-color: #ff4444; }
        .danger .value { color: #ff4444; }
        .section { background: #1e1e1e; padding: 20px; border-radius: 8px; margin-bottom: 20px; }
        .section h2 { margin-top: 0; border-bottom: 1px solid #333; padding-bottom: 10px; color: #00ffcc; }
        .log-box { background: #000; padding: 15px; border-radius: 5px; height: 300px; overflow-y: auto; font-family: monospace; font-size: 14px; color: #00ff00; }
        .log-box p { margin: 5px 0; border-bottom: 1px dashed #333; padding-bottom: 3px; }
        .gem-tag { display: inline-block; background: #333; padding: 5px 10px; border-radius: 20px; margin: 5px; font-size: 14px; border: 1px solid #555; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🛡️ NEXUS V37 DASHBOARD</h1>
            <p>Trạng thái tự động cập nhật sau mỗi 5 giây</p>
        </div>
        
        <div class="stats-grid">
            <div class="stat-card">
                <h3>⏱️ Uptime</h3>
                <div class="value">{{ uptime }}</div>
            </div>
            <div class="stat-card">
                <h3>⚔️ Hunts</h3>
                <div class="value">{{ stats['hunts'] }}</div>
            </div>
            <div class="stat-card">
                <h3>🛡️ Battles</h3>
                <div class="value">{{ stats['battles'] }}</div>
            </div>
            <div class="stat-card danger">
                <h3>🚨 Captcha gặp</h3>
                <div class="value">{{ stats['captchas'] }}</div>
            </div>
        </div>

        <div class="section">
            <h2>💎 Ngọc đã trang bị</h2>
            <div>
                {% if stats['gems_used'] %}
                    {% for gem_id, count in stats['gems_used'].items() %}
                        <span class="gem-tag">Gem ID: {{ gem_id }} (Đã dùng: {{ count }} lần)</span>
                    {% endfor %}
                {% else %}
                    <span style="color: #888;">Chưa sử dụng viên ngọc nào.</span>
                {% endif %}
            </div>
        </div>

        <div class="section">
            <h2>📜 Lịch sử hoạt động (Live Logs)</h2>
            <div class="log-box">
                {% for log in stats['logs'] %}
                    <p>{{ log }}</p>
                {% endfor %}
                {% if not stats['logs'] %}
                    <p>Hệ thống đang chờ dữ liệu...</p>
                {% endif %}
            </div>
        </div>
    </div>
</body>
</html>
"""

@app.route('/')
def dashboard():
    # Tính thời gian chạy (Uptime)
    uptime_seconds = int(time.time() - start_time)
    uptime_str = str(timedelta(seconds=uptime_seconds))
    return render_template_string(HTML_TEMPLATE, uptime=uptime_str, stats=stats)

def run_flask_server():
    # Tắt log mặc định của Flask để tránh rối màn hình console
    import logging
    log = logging.getLogger('werkzeug')
    log.setLevel(logging.ERROR)
    app.run(host='0.0.0.0', port=8080)

# ==================== LOGIC TOOL (BOT) ====================
running = True        
is_paused = False     
hunt_count = 0  
total_actions_in_loop = 0
current_channel_index = 0

waiting_for_inv = False
waiting_for_info = False   
target_hunt_count = 100    
BANNED_GEM_IDS = [56, 57, 70, 71, 77, 78]

DAILY_LIMIT = 8000
daily_command_count = 0
last_reset_time = time.time()

bot = commands.Bot(command_prefix="!", self_bot=True, chunk_guilds_at_startup=False)

def send_webhook_log(msg):
    try: requests.post(DISCORD_WEBHOOK_URL, json={"content": msg}, timeout=5)
    except: pass

async def send_stealth_message(channel, text):
    global is_paused
    if is_paused or not channel: return False
    try:
        async with channel.typing():
            await asyncio.sleep(max(len(text) / 25, 0.2))
            if is_paused: return False 
            await channel.send(text)
            return True
    except: return False

async def infinite_tag_alarm_loop(channel):
    global is_paused, running
    count = 1
    add_log("🚨 BÁO ĐỘNG ĐỎ: Phát hiện Captcha - Tạm dừng Tool!")
    alarm_templates = [
        "🔴 **[HỆ THỐNG CẢNH BÁO CAO CẤP]** 🔴\n> ⚠️ <@{uid}> Nhanh chóng kiểm tra Captcha!\n> 🔄 Trạng thái: **TẠM DỪNG** (#{cnt})",
    ]
    while is_paused and running:
        try:
            final_alarm_msg = random.choice(alarm_templates).format(uid=YOUR_USER_ID, cnt=count)
            await channel.send(final_alarm_msg)
            count += 1
        except: pass
        await asyncio.sleep(random.uniform(4.0, 6.0))

async def check_and_stop_for_captcha(message):
    global is_paused
    if not message or is_paused: return
    
    if message.author.id == 408785106942115842:
        leak_proof_detected = False
        core_keywords = ["captcha", "banned", "verify", "link", "⚠️", "robot", "human", "xác minh", "suspicious"]
        content_raw = message.content.lower() if message.content else ""
        
        if any(word in content_raw for word in core_keywords): leak_proof_detected = True

        if not leak_proof_detected and message.embeds:
            for embed in message.embeds:
                embed_aggregated_text = []
                if embed.title: embed_aggregated_text.append(embed.title.lower())
                if embed.description: embed_aggregated_text.append(embed.description.lower())
                combined_embed_string = " ".join(embed_aggregated_text)
                if any(word in combined_embed_string for word in core_keywords):
                    leak_proof_detected = True; break

        if leak_proof_detected:
            is_paused = True  
            stats["captchas"] += 1 # GHI NHẬN VÀO WEB
            print(f"\n{Colors.FAIL}{Colors.BOLD}[🚨 CODE RED] PHÁT HIỆN BỊ QUÉT! DỪNG TOOL NGAY LẬP TỨC!{Colors.ENDC}")
            send_webhook_log(f"🚨 📞 <@{YOUR_USER_ID}> **[BẢO VỆ V37 WEB] KHÓA LUỒNG FARM KHẨN CẤP!**")
            
            try: await bot.change_presence(status=discord.Status.dnd)
            except: pass
            bot.loop.create_task(infinite_tag_alarm_loop(message.channel))
            return

@bot.event
async def on_message(message):
    global is_paused, running, waiting_for_inv, waiting_for_info, target_hunt_count
    if not running: return
    
    if message.author.id == 408785106942115842 and is_paused:
        content = message.content.lower() if message.content else ""
        if any(ok_word in content for ok_word in ["thank you", "verified", "success", "hoàn thành"]):
            is_paused = False
            add_log("✅ Đã giải Captcha thành công! Khôi phục Farm.")
            print(f"\n{Colors.GREEN}{Colors.BOLD}[▶️ AUTO-RESUME] Giải Captcha xong. Khôi phục farm...{Colors.ENDC}")
            return

    if message.author.id == YOUR_USER_ID and message.content.strip().lower() == "go":
        if is_paused:
            is_paused = False
            add_log("▶️ Chủ nhân đã ra lệnh tiếp tục (GO).")
            print(f"\n{Colors.GREEN}[▶️ RESUME BY OWNER] Nhận lệnh thủ công từ sếp. Tiếp tục cày!{Colors.ENDC}")
            return

    await check_and_stop_for_captcha(message)

    if message.author.id == 408785106942115842 and waiting_for_info:
        is_our_info = False
        if message.embeds:
            author_name = str(message.embeds[0].author.name if message.embeds[0].author else "").lower()
            if "info" in author_name or str(bot.user.name).lower() in author_name:
                is_our_info = True
                
        if is_our_info:
            waiting_for_info = False
            await asyncio.sleep(0.5)
            content = message.content.lower() if message.content else ""
            if message.embeds:
                desc = message.embeds[0].description or ""
                content += " " + desc.lower()
                for f in message.embeds[0].fields:
                    content += f" {f.name} {f.value}".lower()
                    
            uses = re.findall(r'(\d+)\s*uses? left', content)
            if uses:
                min_uses = min([int(u) for u in uses])
                target_hunt_count = max(1, min_uses - 1) 
                add_log(f"📊 Ngọc cạn nhất còn {min_uses} lượt. Đặt mốc: {target_hunt_count}")
            else:
                target_hunt_count = 50

    if message.author.id == 408785106942115842 and waiting_for_inv:
        if message.content and ("inventory" in message.content.lower() or "💎" in message.content or "`id`" in message.content.lower()):
            waiting_for_inv = False
            await asyncio.sleep(0.5)
            found_gems_str = re.findall(r'`id:\s*(\d+)`', message.content.lower())
            if not found_gems_str and message.embeds:
                for embed in message.embeds:
                    desc = embed.description if embed.description else ""
                    found_gems_str.extend(re.findall(r'`id:\s*(\d+)`', desc.lower()))
            
            found_gems = list(set([int(x) for x in found_gems_str]))
            valid_hunt_gems = [x for x in found_gems if 51 <= x <= 57 and x not in BANNED_GEM_IDS]
            valid_x2_gems = [x for x in found_gems if 65 <= x <= 71 and x not in BANNED_GEM_IDS]
            valid_luck_gems = [x for x in found_gems if 72 <= x <= 78 and x not in BANNED_GEM_IDS]

            valid_gems_to_use = []
            if valid_hunt_gems: valid_gems_to_use.append(max(valid_hunt_gems))
            if valid_x2_gems: valid_gems_to_use.append(max(valid_x2_gems))
            if valid_luck_gems: valid_gems_to_use.append(max(valid_luck_gems))

            if valid_gems_to_use:
                gem_ids_string = " ".join(str(g) for g in valid_gems_to_use)
                combined_command = f"owo use {gem_ids_string}"
                
                # CẬP NHẬT GEMS LÊN WEB
                for gid in valid_gems_to_use:
                    stats["gems_used"][str(gid)] = stats["gems_used"].get(str(gid), 0) + 1
                
                add_log(f"💎 Đã tự động dùng ngọc ID: {gem_ids_string}")
                print(f"{Colors.BLUE}[{time.strftime('%H:%M:%S')}] 💎 [AUTO-GEM] Trang bị ngọc: {combined_command}{Colors.ENDC}")
                
                if not is_paused and running:
                    await send_stealth_message(message.channel, combined_command)
                    await asyncio.sleep(1.5)
            
            if not is_paused and running:
                waiting_for_info = True
                await send_stealth_message(message.channel, "owo info")

@bot.event
async def on_ready():
    print(f"{Colors.CYAN}{Colors.BOLD}")
    print(f"╔════════════════════════════════════════════════════════╗")
    print(f"║   🚀 TỰ HÀO OWNER: {OWNER_NAME.ljust(35)} ║")
    print(f"║   🌐 NEXUS V37 WEB DASHBOARD EDITION 🌐                ║")
    print(f"╚════════════════════════════════════════════════════════╝{Colors.ENDC}")
    print(f"{Colors.GREEN} [+] Đăng nhập thành công: {bot.user.name}{Colors.ENDC}")
    print(f"{Colors.BLUE} [+] Trang Web Quản Lý đang mở tại: http://127.0.0.1:8080{Colors.ENDC}")
    print(f"{Colors.WARNING} [!] Mở trình duyệt điện thoại và truy cập link trên nhé!{Colors.ENDC}\n")
    bot.loop.create_task(core_farm_scheduler())
    add_log("🚀 Hệ thống bắt đầu hoạt động thành công.")

async def core_farm_scheduler():
    global running, is_paused, hunt_count, total_actions_in_loop, current_channel_index
    global waiting_for_inv, target_hunt_count, daily_command_count, last_reset_time
    
    await bot.wait_until_ready()
    init_channel = bot.get_channel(FARM_CHANNEL_IDS[0])
    waiting_for_inv = True
    await send_stealth_message(init_channel, "owoinv")
        
    while running:
        current_time = time.time()
        if current_time - last_reset_time >= 86400:
            daily_command_count = 0
            last_reset_time = current_time

        if daily_command_count >= DAILY_LIMIT:
            wait_sec = 86400 - (current_time - last_reset_time)
            if wait_sec > 0: await asyncio.sleep(wait_sec)
            daily_command_count = 0
            last_reset_time = time.time()
            continue

        if is_paused:
            await asyncio.sleep(1)
            continue

        if total_actions_in_loop > 0 and total_actions_in_loop % random.randint(20, 30) == 0:
            current_channel_index = (current_channel_index + 1) % len(FARM_CHANNEL_IDS)

        active_channel = bot.get_channel(FARM_CHANNEL_IDS[current_channel_index])

        if not is_paused and running:
            hunt_cmds = ["owoh", "owo hunt", "owo h"]
            battle_cmds = ["owob", "owo battle", "owo b"]
            
            commands_pool = [random.choice(hunt_cmds), random.choice(battle_cmds)]
            random.shuffle(commands_pool)

            first_cmd = commands_pool[0]
            success1 = await send_stealth_message(active_channel, first_cmd)
            if "h" in first_cmd or "hunt" in first_cmd: stats["hunts"] += 1
            else: stats["battles"] += 1
            
            await asyncio.sleep(random.uniform(0.7, 1.2)) 

            second_cmd = commands_pool[1]
            success2 = await send_stealth_message(active_channel, second_cmd)
            if "h" in second_cmd or "hunt" in second_cmd: stats["hunts"] += 1
            else: stats["battles"] += 1
            
            if success1 or success2:
                hunt_count += 1
                total_actions_in_loop += 2
                daily_command_count += 2  
                
                log_text = f"Thực thi thành công: {first_cmd} & {second_cmd} (Lượt thứ: {hunt_count})"
                add_log(log_text)
                print(f"{Colors.GREEN}[{time.strftime('%H:%M:%S')}] {Colors.ENDC}[Kênh {current_channel_index + 1}] ⚔️ {Colors.CYAN}{first_cmd}{Colors.ENDC} -> 🛡️ {Colors.CYAN}{second_cmd}{Colors.ENDC} {Colors.WARNING}[{hunt_count}/{target_hunt_count}]{Colors.ENDC}")

            if running and not is_paused:
                await asyncio.sleep(random.uniform(0.8, 1.5)) 
                chosen_type = random.choices(["owo", "uwu"], weights=[50, 50], k=1)[0]
                final_msg = random.choice([chosen_type, chosen_type.capitalize(), chosen_type.upper()])
                await send_stealth_message(active_channel, final_msg)

        if hunt_count >= target_hunt_count and not is_paused and running:
            add_log("📦 Cạn ngọc, đang khởi động lấy ngọc mới...")
            waiting_for_inv = True
            await send_stealth_message(active_channel, "owoinv")
            hunt_count = 0 

        if running and not is_paused:
            await asyncio.sleep(15.1)

if __name__ == "__main__":
    # Khởi chạy máy chủ Web ngầm (Thread)
    Thread(target=run_flask_server, daemon=True).start()
    
    # Khởi chạy Bot Discord
    try: 
        bot.run(TOKEN_TAI_KHOAN_FARM)
    except discord.LoginFailure: 
        print(f"{Colors.FAIL}[Lỗi] Token không hợp lệ!{Colors.ENDC}")
