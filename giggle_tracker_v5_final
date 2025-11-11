# giggle_tracker_v5_final.py
import requests
import time
from datetime import datetime
import threading
from flask import Flask, render_template_string
import winsound
import os

# =================== CONFIG ===================
API_KEY = "FBVZ6FPYH5AZPM8D6M4617C3N5RM2DXNFJ"
POLL_INTERVAL = 15
MIN_VALUE_USD = 10.0  # FILTRO: SOLO > $10 USD

# TELEGRAM: Render o local (con valores por defecto)
TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN", "8563562493:AAGiuzs7N_w63tLAEV6T_wXsDIcQ-cprI8Y")
TELEGRAM_CHAT_ID = os.getenv("TELEGRAM_CHAT_ID", "1063182207")
TELEGRAM_API = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"

# WALLETS A SEGUIR
WALLETS = [
    {"address": "0x61032a6D4D8a18964F4D2885439437F260f58aD6", "name": "GIGGLE0"},
    {"address": "0xaf25627aC5a3ac2EFC3B18bc4FC4E4E650F803Dc", "name": "GIGGLE1"},
    {"address": "0xb95Aef34715696Bfb80B8Df98d3ad75742EB4947", "name": "TRAIDER"},
    {"address": "0x9b6C02B5E62e979946a2b94EEb787C05b22d24B2", "name": "GIGGLE GRANDE"},
]

# LOGS
LOG_FILE = "giggle_tracker_v5_final.log"
if not os.path.exists(LOG_FILE):
    with open(LOG_FILE, "w", encoding="utf-8") as f:
        f.write(f"[{datetime.now()}] GIGGLE TRACKER v5 + RENDER INICIADO\n")

def log(msg):
    print(msg)
    with open(LOG_FILE, "a", encoding="utf-8") as f:
        f.write(f"[{datetime.now().strftime('%H:%M:%S')}] {msg}\n")

def play_sound():
    try: winsound.Beep(3000, 400)
    except: pass

def send_telegram(msg):
    try:
        requests.post(TELEGRAM_API, data={
            "chat_id": TELEGRAM_CHAT_ID,
            "text": msg,
            "parse_mode": "Markdown",
            "disable_web_page_preview": True
        }, timeout=5)
        # play_sound()  # Desactivado en Render (no hay sonido)
        log(f"ALERTA >$10 → {msg.splitlines()[0][:60]}...")
    except Exception as e:
        log(f"Error Telegram: {e}")

# =================== PRECIO BNB ===================
BNB_PRICE = 600
def update_bnb_price():
    global BNB_PRICE
    try:
        r = requests.get("https://api.binance.com/api/v3/ticker/price?symbol=BNBUSDT", timeout=5)
        BNB_PRICE = float(r.json()["price"])
    except: pass

# =================== BSCSCAN API ===================
def get_txs(address, action):
    params = {
        "module": "account",
        "action": action,
        "address": address,
        "page": 1,
        "offset": 50,
        "sort": "desc",
        "apikey": API_KEY
    }
    try:
        r = requests.get("https://api.bscscan.com/api", params=params, timeout=10)
        data = r.json()
        if data.get("status") == "1" and isinstance(data.get("result"), list):
            return data["result"]
    except: pass
    return []

# =================== DASHBOARD ===================
dashboard_data = {
    "last_update": "",
    "transactions": [],
    "bnb_price": 0
}

last_seen = {w["address"].lower(): 0 for w in WALLETS}

# DEX (TODOS LOS ACTIVOS + DRAGUN69)
DEX_CONTRACTS = {
    "0x10ed43c718714eb63d5aa57b78b54704e256024e": "PancakeSwap V2",
    "0x1b96b92314c44b159149f7e913fcff7957679990": "PancakeSwap V3",
    "0x3a6d8ca21d1cf76f653a67577fa0d27453350dd8": "Biswap",
    "0xc0788a3ad43d79aa53b09c2eacc88a2512479d8d": "ApeSwap",
    "0x9333c74bdd1e118634fe5664aca7a9710b108bab": "OKX DEX Router 5",
    "0x6015126d7d23648c2e4466693b8deab005ffaba8": "OKX DEX Router 6",
    "0xc44ad35b5a41c428c0eae842f20f84d1ff6ed917": "OKX Dex Router (General)",
    "0xca980f000771f70b15647069e9e541ef73f71f2f": "Dragun69 Router",
}

# =================== MAIN LOOP ===================
def monitor_loop():
    global last_seen
    update_bnb_price()
    dashboard_data["bnb_price"] = f"{BNB_PRICE:.2f}"

    while True:
        try:
            update_bnb_price()
            dashboard_data["last_update"] = datetime.now().strftime("%H:%M:%S")
            activity = False

            for wallet in WALLETS:
                addr = wallet["address"].lower()
                name = wallet["name"]

                txs_normal = get_txs(wallet["address"], "txlist")
                txs_token = get_txs(wallet["address"], "tokentx")
                all_txs = txs_normal + txs_token

                new_txs = [
                    tx for tx in all_txs
                    if isinstance(tx, dict) and "timeStamp" in tx
                    and int(tx["timeStamp"]) > last_seen[addr]
                ]

                if not new_txs: continue

                new_txs.sort(key=lambda x: int(x["timeStamp"]))
                activity = True

                for tx in new_txs:
                    try:
                        hash_tx = tx.get("hash", "N/A")
                        timestamp = int(tx.get("timeStamp", 0))
                        if timestamp <= last_seen[addr]: continue

                        is_token_tx = "tokenSymbol" in tx
                        value_raw = int(tx.get("value", 0))
                        decimals = 18 if not is_token_tx else int(tx.get("tokenDecimal", 18))
                        value = value_raw / (10 ** decimals)
                        token = tx.get("tokenSymbol", "BNB")
                        usd = value * BNB_PRICE

                        # FILTRO: SOLO SI > $10
                        if usd < MIN_VALUE_USD:
                            continue  # Ignora transacciones pequeñas

                        to_addr = tx.get("to", "")[:10].lower()
                        from_addr = tx.get("from", "").lower()

                        # DETECCIÓN DE DEX
                        dex_name = "Contrato"
                        if to_addr in DEX_CONTRACTS:
                            dex_name = DEX_CONTRACTS[to_addr]
                        elif from_addr in DEX_CONTRACTS:
                            dex_name = DEX_CONTRACTS[from_addr]

                        action = "ENVIÓ" if from_addr == addr else "RECIBIÓ"

                        # MENSAJE TELEGRAM
                        if is_token_tx:
                            msg = f"""*{name} → {action}*
`{value:,.6f} {token}`
~${usd:,.0f}
A: `{to_addr}...`
*{dex_name}*
[Ver TX](https://bscscan.com/tx/{hash_tx})"""
                        else:
                            msg = f"""*{name} → {action} BNB*
`{value:.6f} BNB` → ~${usd:,.0f}
A: `{to_addr}...`
[Ver TX](https://bscscan.com/tx/{hash_tx})"""

                        send_telegram(msg)

                        # DASHBOARD
                        dashboard_data["transactions"].insert(0, {
                            "wallet": name,
                            "time": datetime.fromtimestamp(timestamp).strftime("%H:%M:%S"),
                            "action": action,
                            "amount": f"{value:,.6f} {token}",
                            "usd": f"${usd:,.0f}",
                            "to": to_addr,
                            "hash": hash_tx,
                            "dex": dex_name
                        })
                        if len(dashboard_data["transactions"]) > 60:
                            dashboard_data["transactions"] = dashboard_data["transactions"][:60]

                        last_seen[addr] = timestamp

                    except Exception as e:
                        log(f"Error TX {name}: {e}")

            time.sleep(1 if activity else POLL_INTERVAL)

        except Exception as e:
            log(f"Error loop: {e}")
            time.sleep(POLL_INTERVAL)

# =================== DASHBOARD ===================
app = Flask(__name__)

HTML = '''
<!DOCTYPE html>
<html><head><meta charset="UTF-8"><meta http-equiv="refresh" content="12">
<title>GIGGLE TRACKER v5 + RENDER</title>
<style>
    body { font-family: 'Courier New'; background: #000; color: #0f0; margin: 0; padding: 20px; }
    h1 { color: #0f0; text-align: center; text-shadow: 0 0 12px #0f0; font-size: 2.5em; margin-bottom: 10px; }
    .stats { background: #111; padding: 12px; border: 2px solid #0f0; border-radius: 10px; text-align: center; font-size: 1.1em; }
    table { width: 100%; border: 2px solid #0f0; background: #111; margin-top: 15px; }
    th { background: #222; padding: 10px; color: #0f0; font-weight: bold; }
    td { padding: 8px; border-bottom: 1px dashed #0f0; }
    a { color: #0ff; font-weight: bold; text-decoration: none; }
    .wallet { font-weight: bold; color: #0ff; }
    .update { color: #888; }
</style></head>
<body>
<h1>GIGGLE TRACKER v5 + RENDER 24/7</h1>
<div class="stats">
    <b>BNB:</b> ${{ bnb_price }} | 
    <b>Filtro:</b> > $10 | 
    <b>Última actualización:</b> <span class="update">{{ last_update }}</span>
</div>
<table>
    <tr><th>Wallet</th><th>Hora</th><th>Acción</th><th>Monto</th><th>USD</th><th>Destino</th><th>DEX</th><th>TX</th></tr>
    {% for t in transactions %}
    <tr>
        <td class="wallet">{{ t.wallet }}</td>
        <td>{{ t.time }}</td>
        <td>{{ t.action }}</td>
        <td>{{ t.amount }}</td>
        <td>{{ t.usd }}</td>
        <td>{{ t.to }}...</td>
        <td>{{ t.dex }}</td>
        <td><a href="https://bscscan.com/tx/{{ t.hash }}" target="_blank">VER</a></td>
    </tr>
    {% endfor %}
</table>
</body></html>
'''

@app.route('/')
def dashboard():
    return render_template_string(HTML,
        bnb_price=dashboard_data["bnb_price"],
        last_update=dashboard_data["last_update"],
        transactions=dashboard_data["transactions"]
    )

# =================== INICIO ===================
if __name__ == '__main__':
    log("GIGGLE TRACKER v5 + RENDER INICIADO")
    wallet_list = "\n".join([f"• *{w['name']}*: `{w['address']}`" for w in WALLETS])
    send_telegram(f"*GIGGLE TRACKER v5 + RENDER 24/7 INICIADO*\n\n**Wallets:**\n{wallet_list}\n\n**Filtro activo:** > $10 USD\n\n[Dashboard](http://localhost:5000)")

    threading.Thread(target=monitor_loop, daemon=True).start()
    
    # Render: Puerto dinámico
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port, debug=False, use_reloader=False)
