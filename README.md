# AI-pellet-boiler-program
universal AI program for adjusting pellet boilers
#!/usr/bin/env python3
# ==============================================
# AI OPTIMIZATION FOR PELLET BOILER (RPi)
# SAFE + SLOW + INDUSTRIAL STYLE
# ==============================================

import serial, json, time, os
from collections import deque
from influxdb import InfluxDBClient

# ==============================================
# CONFIG
# ==============================================
SERIAL_PORT = "/dev/ttyUSB0"
BAUDRATE = 115200

WATCHDOG_TIMEOUT = 10
LEARNING_RATE = 0.02

FAN_MIN, FAN_MAX = 400, 1023
FEED_MIN, FEED_MAX = 0.30, 1.00

STATE_FILE = "ai_state.json"

# ==============================================
# DATABASE
# ==============================================
db = InfluxDBClient(host="localhost", port=8086)
db.switch_database("pellet")

# ==============================================
# SERIAL
# ==============================================
ser = serial.Serial(SERIAL_PORT, BAUDRATE, timeout=1)

# ==============================================
# AI STATE
# ==============================================
k_fan = 1.0
k_feed = 1.0
last_fan = 700

# temperature history for derivative
temp_hist = deque(maxlen=10)
time_hist = deque(maxlen=10)

# ==============================================
# UTILS
# ==============================================
def clamp(v, lo, hi):
    return max(lo, min(hi, v))

def norm(x, a, b):
    if b <= a:
        return 0.0
    return clamp((x - a) / (b - a), 0.0, 1.0)

# ==============================================
# STATE SAVE / LOAD
# ==============================================
def load_state():
    global k_fan, k_feed
    if os.path.exists(STATE_FILE):
        with open(STATE_FILE) as f:
            s = json.load(f)
            k_fan = s.get("k_fan", 1.0)
            k_feed = s.get("k_feed", 1.0)

def save_state():
    with open(STATE_FILE, "w") as f:
        json.dump({"k_fan": k_fan, "k_feed": k_feed}, f)

# ==============================================
# DATA ANALYSIS
# ==============================================
def get_dT_dt():
    if len(temp_hist) < 2:
        return 0.0
    dt = time_hist[-1] - time_hist[0]
    if dt <= 0:
        return 0.0
    return (temp_hist[-1] - temp_hist[0]) / (dt / 60)

def compute_score(T_dim, dT_dt, fan):
    return (
        0.45 * norm(T_dim, 90, 160) +
        0.35 * norm(dT_dt, 0.01, 0.07) +
        0.20 * (1 - norm(fan, 500, 1000))
    )

# ==============================================
# LEARNING
# ==============================================
def adapt(score):
    global k_fan, k_feed

    if score > 0.75:
        k_feed *= (1 - LEARNING_RATE)
        k_fan  *= (1 - LEARNING_RATE / 2)
    elif score < 0.4:
        k_feed *= (1 + LEARNING_RATE)
        k_fan  *= (1 + LEARNING_RATE / 2)

    k_feed = clamp(k_feed, 0.85, 1.15)
    k_fan  = clamp(k_fan,  0.90, 1.10)

# ==============================================
# AI DECISION
# ==============================================
def ai_predict(data):
    delta = data["T_dim"] - data["T_zg_top"]

    if delta > 20:
        fan, feed = 1000, 0.9
    elif delta > 10:
        fan, feed = 800, 0.6
    else:
        fan, feed = 600, 0.4

    fan  = int(clamp(fan * k_fan, FAN_MIN, FAN_MAX))
    feed = round(clamp(feed * k_feed, FEED_MIN, FEED_MAX), 2)

    return fan, feed

# ==============================================
# LOGGING
# ==============================================
def log_to_db(data, fan, feed, score):
    db.write_points([{
        "measurement": "boiler",
        "fields": {
            "T_pec": data["T_pec"],
            "T_dim": data["T_dim"],
            "T_zg_top": data["T_zg_top"],
            "T_zg_bot": data["T_zg_bot"],
            "fan": fan,
            "feed": feed,
            "score": score,
            "fault": data["fault"]
        }
    }])

# ==============================================
# MAIN LOOP
# ==============================================
print("🔥 Pellet AI started (SAFE MODE)")
load_state()

last_rx = time.time()

while True:
    try:
        if time.time() - last_rx > WATCHDOG_TIMEOUT:
            print("⚠️ ESP32 timeout")
            time.sleep(1)
            continue

        line = ser.readline().decode().strip()
        if not line:
            continue

        data = json.loads(line)
        last_rx = time.time()

        if data["fault"] == 1:
            continue

        temp_hist.append(data["T_pec"])
        time_hist.append(time.time())

        dT_dt = get_dT_dt()
        score = compute_score(data["T_dim"], dT_dt, last_fan)

        adapt(score)
        fan, feed = ai_predict(data)
        last_fan = fan

        log_to_db(data, fan, feed, score)
        save_state()

        ser.write((json.dumps({
            "fan": fan,
            "feed": feed,
            "valid": 1,
            "hb": data["hb"]
        }) + "\n").encode())

    except Exception as e:
        print("AI ERROR:", e)
        time.sleep(1)
