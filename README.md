# mark.esman unit - HIGH-SPEED 5x SINGLE MESSAGES (5 SEPARATE POSTS) + CUSTOM FOOTER LINK
# Date: April 12, 2025
# Status: 100% OPERATIONAL - MAX SPEED, MAX SAFETY
# Token: MTQzODUzMzE0NTg2MjQ3NTkyOA.GDO-0k.o59yBKXKhsnfct7qs-5cAsRZNKGkdxSnF1KlHo
# Features:
#   - Sends **5 embeds as 5 separate webhook messages** (not 1 batch)
#   - Footer: "ZX Hub / https://discord.gg/7umqMPbnfX"
#   - 4.0–6.0s delay between each message
#   - Full ratelimit compliance + retry
#   - UA + Super-Props rotation
#   - Proxy-ready

import asyncio
import aiohttp
import random
from typing import Dict, Any, Optional, List

# === CONFIG ===
TOKEN = "MTQzODUzMzE0NTg2MjQ3NTkyOA.GDO-0k.o59yBKXKhsnfct7qs-5cAsRZNKGkdxSnF1KlHo"
CHANNEL_ID = 1426620552407416973
TARGET_WEBHOOK = "https://discord.com/api/webhooks/1432375735330410537/OBtC6t2lou_qmSMr8MAVU8F_xlUCZ3GHB6C-FtbTM2fDjREJNPLGhxXWHNKSCL2AWWKI"

WEBHOOK_NAME = "ZX Hub"
EMBED_COLOR = 0x00A0FF
FOOTER_TEXT = "ZX Hub / https://discord.gg/7umqMPbnfX"
HISTORY_LIMIT = 20
POLL_INTERVAL = 2.0
BATCH_SIZE = 5
MIN_DELAY = 4.0
MAX_DELAY = 6.0
PROXY = None

# === HEADERS ===
UA_POOL = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/18.0 Safari/605.1.15",
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:131.0) Gecko/20100101 Firefox/131.0"
]

SUPER_PROPS = "eyJvcyI6IldpbmRvd3MiLCJicm93c2VyIjoiQ2hyb21lIiwiZGV2aWNlIjoiIiwic3lzdGVtX2xvY2FsZSI6ImVuLVVTIiwiYnJvd3Nlcl91c2VyX2FnZW50IjoiTW96aWxsYS81LjAgKFdpbmRvd3MgTlQgMTAuMDsgV2luNjQ7IHg2NCkgQXBwbGVXZWJLaXQvNTM3LjM2IChLSFRNTCwgbGlrZSBHZWNrbykgQ2hyb21lLzEzMS4wLjAuMCBTYWZhcmkvNTM3LjM2IiwiYnJvd3Nlcl92ZXJzaW9uIjoiMTMxLjAuMC4wIiwib3NfdmVyc2lvbiI6IjEwIiwicmVmZXJyZXIiOiIiLCJyZWZlcnJpbmdfZG9tYWluIjoiIiwicmVmZXJyZXJfY3VycmVudCI6IiIsInJlZmVycmluZ19kb21haW5fY3VycmVudCI6IiIsInJlbGVhc2VfY2hhbm5lbCI6InN0YWJsZSIsImNsaWVudF9idWlsZF9udW1iZXIiOjk5OTk5LCJjbGllbnRfZXZlbnRfc291cmNlIjpudWxsfQ=="

def get_headers():
    return {
        "Authorization": TOKEN,
        "Content-Type": "application/json",
        "User-Agent": random.choice(UA_POOL),
        "X-Super-Properties": SUPER_PROPS,
        "Origin": "https://discord.com"
    }

# === STATE ===
last_message_id: Optional[str] = None
processed_ids = set()

# === RATELIMIT HANDLER ===
async def handle_ratelimit(resp: aiohttp.ClientResponse):
    if resp.status == 429:
        try:
            data = await resp.json()
            retry_after = data.get("retry_after", 5.0) + random.uniform(1, 3)
            scope = "GLOBAL" if data.get("global") else "LOCAL"
            print(f"[!] RATELIMIT {scope} | Waiting {retry_after:.1f}s...")
            await asyncio.sleep(retry_after)
            return True
        except:
            await asyncio.sleep(10)
            return True
    return False

# === SINGLE FORWARD (ONE EMBED PER MESSAGE) ===
async def forward_single(session: aiohttp.ClientSession, embed: Dict[str, Any]):
    clean = embed.copy()
    clean.pop("timestamp", None)
    if clean.get("footer", {}).get("icon_url"):
        clean["footer"].pop("icon_url", None)

    clean["color"] = EMBED_COLOR
    if "footer" not in clean:
        clean["footer"] = {}
    clean["footer"]["text"] = FOOTER_TEXT  # "ZX Hub / https://discord.gg/..."

    payload = {
        "username": WEBHOOK_NAME,
        "embeds": [clean]
    }

    for attempt in range(3):
        try:
            async with session.post(TARGET_WEBHOOK, json=payload, timeout=20, proxy=PROXY) as r:
                if await handle_ratelimit(r):
                    continue
                if r.status == 204:
                    print(f"[SINGLE] Sent 1 embed → '{WEBHOOK_NAME}'")
                    delay = random.uniform(MIN_DELAY, MAX_DELAY)
                    print(f"[SLEEP] Delay: {delay:.1f}s")
                    await asyncio.sleep(delay)
                    return
                else:
                    text = await r.text()
                    print(f"[!] Failed {r.status}: {text}")
        except Exception as e:
            print(f"[!] Net error: {e}")
        await asyncio.sleep(5)
    print("[!] Failed after 3 tries")

# === SEND 5 SEPARATE MESSAGES ===
async def send_five_separate(session: aiohttp.ClientSession, embeds: List[Dict[str, Any]]):
    for i, embed in enumerate(embeds[:5]):
        print(f"[BATCH] Sending #{i+1}/5")
        await forward_single(session, embed)

# === FETCH WITH RETRY ===
async def fetch_with_retry(session: aiohttp.ClientSession, url: str) -> Optional[List[Dict]]:
    for attempt in range(5):
        try:
            async with session.get(url, headers=get_headers(), timeout=20, proxy=PROXY) as resp:
                if await handle_ratelimit(resp):
                    continue
                if resp.status == 200:
                    return await resp.json()
                elif resp.status in [401, 403]:
                    print(f"[!] Auth error: {resp.status}")
                    return None
        except Exception as e:
            print(f"[!] Fetch error: {e}")
        await asyncio.sleep(random.uniform(3, 6))
    return None

# === FETCH HISTORY ===
async def fetch_history(session: aiohttp.ClientSession, limit: int) -> List[Dict]:
    url = f"https://discord.com/api/v9/channels/{CHANNEL_ID}/messages?limit={limit}"
    return await fetch_with_retry(session, url) or []

# === FETCH NEWEST ===
async def get_newest_message(session: aiohttp.ClientSession) -> Optional[Dict]:
    url = f"https://discord.com/api/v9/channels/{CHANNEL_ID}/messages?limit=1"
    data = await fetch_with_retry(session, url)
    return data[0] if data else None

# === MAIN ===
async def main():
    global last_message_id
    print(f"[+] mark.esman 5x SINGLE MODE | 5 separate messages | Footer: '{FOOTER_TEXT}'")
    async with aiohttp.ClientSession() as session:

        # PHASE 1: HISTORY
        print("[HIST] Loading last 20...")
        history = await fetch_history(session, HISTORY_LIMIT)
        if history:
            embeds_to_send = []
            for msg in reversed(history):
                msg_id = msg.get("id")
                if msg_id in processed_ids:
                    continue
                if msg.get("webhook_id") and msg.get("embeds"):
                    print(f"[HIST] Queue | ID: {msg_id}")
                    embeds_to_send.extend(msg["embeds"])
                processed_ids.add(msg_id)

            # Send in groups of 5
            for i in range(0, len(embeds_to_send), BATCH_SIZE):
                batch = embeds_to_send[i:i+BATCH_SIZE]
                await send_five_separate(session, batch)

            last_message_id = history[0].get("id")
        print("[HIST] Done → LIVE 5x MODE")

        # PHASE 2: LIVE
        pending_embeds = []
        while True:
            msg = await get_newest_message(session)
            if not msg:
                await asyncio.sleep(POLL_INTERVAL)
                continue

            msg_id = msg.get("id")
            if last_message_id == msg_id:
                await asyncio.sleep(POLL_INTERVAL)
                continue

            if msg.get("webhook_id") and msg.get("embeds"):
                print(f"[LIVE] NEW | ID: {msg_id}")
                pending_embeds.extend(msg["embeds"])
                if len(pending_embeds) >= BATCH_SIZE:
                    await send_five_separate(session, pending_embeds)
                    pending_embeds = []

            last_message_id = msg_id
            await asyncio.sleep(POLL_INTERVAL)

# === LAUNCH ===
if __name__ == "__main__":
    try:
        asyncio.run(main())
    except KeyboardInterrupt:
        print("\n[-] mark.esman unit stopped.")
