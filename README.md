# 💧 Infusi-Metrics — Smart IV Fluid Monitoring & Nurse Alert System

A simulated smart IV-drip monitor built on an **Arduino Mega 2560** in the [Wokwi](https://wokwi.com) simulator, with a live **Streamlit dashboard** on the PC. The system watches the IV fluid level, predicts when the bottle will run out, detects leaks and blockages, lets the patient call a nurse, and auto-clamps the line if a low bottle is ignored.

> ⚠️ **Simulation only — not for clinical use or real patient monitoring.**

---

## 📸 Screenshots

### Circuit diagram (Wokwi)
![Circuit Diagram](docs/circuit-diagram.png)

### Streamlit dashboard
![Dashboard – Overview](docs/dashboard-1.png)
![Dashboard – Nurse alerts, commands and ML estimate](docs/dashboard-2.png)
![Dashboard – Status, history, alerts and events](docs/dashboard-3.png)

---

## ✨ Features

| Area | What it does |
|---|---|
| **Fluid level monitoring** | Potentiometer on `A0` simulates the IV bag level (0–100 %), averaged over 8 readings. |
| **Low-level alert** | Below 20 % → red LED, patient bulb, buzzer, and "Refill Needed!" on the LCD. |
| **Auto-clamp** | If the bottle stays low for 60 s (20 s for demos), a servo closes the IV line. Releases automatically on refill. |
| **On-device ML** | Online linear regression over a 60 s window gives the **drain rate (%/min)** and **time-to-empty / time-to-low**. |
| **Early warning** | Chirps and alerts when LOW is predicted within 2 minutes. |
| **Leak & blockage detection** | Compares the short-term rate against a learned baseline: > 2.5× = suspected leak; near zero = suspected blockage. |
| **Refill detection** | A sudden jump of ≥ 10 % is treated as a bottle replacement; the window restarts and keeps the learned baseline. |
| **Nurse call** | Patient button on pin `22`. Unanswered after 30 s → **escalated to backup staff**. Cleared by the reset button or the `ack` command. |
| **Environment sensing** | DHT22 gives room temperature and humidity. |
| **Distance sensing** | HC-SR04 ultrasonic gives distance in cm. |
| **Status displays** | 16×2 I2C LCD, 128×64 SSD1306 OLED, and an RGB status LED. |
| **SD card logging** | `ivlog.csv` (events) and `ivdata.csv` (level samples). |
| **GPS (optional)** | TinyGPS++ reads a GPS module; falls back to a fixed bed location if there's no fix. |
| **Text commands (NLP)** | Plain-English commands over Serial, e.g. `status`, `how long is left`, `where is the patient`, `ack`. |
| **Web dashboard** | Live metrics, nurse alert panel, command buttons, ML next-level estimate, question box, history chart, alerts and events. |
| **Demo simulation modes** | Simulate normal drain, leak, blockage or refill without touching the potentiometer. |

### RGB status light

| Colour | Meaning |
|---|---|
| 🟢 Green | Normal |
| 🟡 Amber | Warning (early warning / leak / blockage) |
| 🔴 Red | LOW level or line clamped |
| 🔵 Blue (blinking) | Nurse call waiting |
| 🔴 Red (fast blink) | Nurse call unanswered — escalated |

---

## 🧰 Hardware (simulated in Wokwi)

- Arduino Mega 2560
- 16×2 LCD with I2C backpack (`0x27`)
- SSD1306 OLED, 128×64 (I2C `0x3C`)
- Potentiometer — IV fluid level sensor
- SG90 servo — IV line clamp
- DHT22 — temperature & humidity
- HC-SR04 — ultrasonic distance sensor
- microSD card module
- Push buttons — nurse call and nurse reset
- Buzzer, red/green LEDs, patient bulb, RGB LED, resistors

### Pin map

| Pin | Connected to |
|---|---|
| `A0` | Fluid sensor (potentiometer) |
| `2` | Patient bulb |
| `3` / `5` | GPS TX / GPS RX (optional) |
| `4` | SD card CS |
| `6` | Servo clamp |
| `7` | Reset / silence button |
| `8` | Buzzer |
| `9` / `10` | Red LED / Green LED |
| `20` / `21` | I2C SDA / SCL (LCD + OLED) |
| `22` | Nurse-call button (internal pull-up) |
| `23` | DHT22 data |
| `24` / `25` / `26` | RGB LED R / G / B |
| `27` / `28` | HC-SR04 TRIG / ECHO |
| `50` / `51` / `52` | SPI (SD card) |

---

## 📁 Project structure

```
Infusi-Metrics/
├── sketch/
│   ├── sketch.ino                    # Arduino firmware
│   └── sketch.ino.mega.hex           # Compiled firmware used by Wokwi
├── diagram.json                      # Wokwi circuit
├── wokwi.toml                        # Wokwi config (RFC2217 serial on port 4000)
├── libraries.txt                     # Arduino libraries
├── dashboard.py                      # Streamlit dashboard
├── smart_features.py                 # ML prediction + question answering
├── searil.reader.py                  # Simple serial reader for testing
└── docs/                             # Screenshots used in this README
```

---

## 🔌 Serial protocol

The Arduino sends one message per line; the dashboard parses them.

| Line | Meaning |
|---|---|
| `DATA,elapsed_ms,level` | Level sample (every 5 s) |
| `EVENT,elapsed_ms,level,event,rate,lat,lon` | Logged event |
| `ENV,temp_C,humidity_%` | DHT22 reading |
| `DIST,distance_cm` | Ultrasonic reading |
| `NURSE,0 / 1 / 2` | 0 = acknowledged, 1 = call active, 2 = unanswered / escalated |
| `STATUS \| Level: … \| Rate: … \| Low: … \| Clamped: … \| Sim: …` | Debug status line |
| `ALERT Bed N: …` | Human-readable alert sentence |

**Event names:** `LOW`, `REFILLED`, `AUTO_CLAMPED`, `EARLY_WARN`, `LEAK_SUSPECTED`, `BLOCK_SUSPECTED`, `NURSE_CALL`, `NURSE_ACK`, `NURSE_ESCALATED`

### Supported text commands

`status` · `how long is left` · `where is the patient` · `silence alarm` · `is the clamp on` · `temperature` / `humidity` / `distance` · `ack` · `simulate drain | leak | block | off` · `refill` · `help`

Add `bed N` to address a specific bed (this monitor serves **Bed 1**).

---

## 🚀 Getting started

### 1. Requirements

- [Visual Studio Code](https://code.visualstudio.com/) with the **Wokwi Simulator** extension (or the Wokwi CLI)
- Python 3.9+

### 2. Install Python packages

```bash
pip install streamlit streamlit-autorefresh pandas pyserial scikit-learn
```

### 3. Start the Wokwi simulation

1. Open the project folder in VS Code.
2. Make sure `wokwi.toml` points to `sketch/sketch.ino.mega.hex`.
3. Press **F1 → Wokwi: Start Simulator**.

The simulator exposes the Arduino serial port at `rfc2217://localhost:4000`.

> To rebuild the firmware, open `sketch/sketch.ino` in the Arduino IDE (board: **Arduino Mega 2560**) and install the libraries in `libraries.txt`, plus *Adafruit Unified Sensor*.

### 4. Run the dashboard

In a separate terminal:

```bash
streamlit run dashboard.py
```

Open the URL Streamlit prints (usually http://localhost:8501).

### 5. (Optional) Test with the serial reader

```bash
python searil.reader.py
```

Use only one client at a time — the Wokwi port accepts a single connection, so stop this before starting the dashboard.

---

## 🧪 Quick demo

1. Start the Wokwi simulation, then the dashboard.
2. Click **Simulate drain** on the dashboard and watch the level, rate and ML estimate update.
3. Click **Simulate leak** or **Simulate blockage** to trigger anomaly alerts.
4. Press the **Nurse Call** button in Wokwi. After 30 s without an acknowledgement, it escalates.
5. Click **Acknowledge nurse call** or type `ack` to clear it.
6. Let the level stay under 20 % to see the servo auto-clamp, then click **Refill bottle**.

For quicker demos, set `CLAMP_DELAY_MS = 20000UL` and `NURSE_ESCALATE_MS = 15000UL` in the sketch.

---

## 🛠️ Troubleshooting

| Problem | Fix |
|---|---|
| `Serial error: Could not open port rfc2217://localhost:4000 … refused` | The Wokwi simulation isn't running. Start it first; the dashboard reconnects every 3 s. |
| Dashboard shows `—` for all readings | No data yet — wait for the simulator to send its first `DATA` / `ENV` lines. |
| Port in use / no data | Another program (e.g. `searil.reader.py`) is connected. Close it. |
| LCD is blank | Try I2C address `0x3F` instead of `0x27` in the sketch. |
| ML estimate says "Collect at least 5 readings" | Wait about 25 s for enough samples. |

---

## 🧠 How the ML works

- **Drain rate** — least-squares slope of the last 12 readings (60 s), converted to %/min.
- **Prediction** — `time_to_empty = level / rate`, `time_to_low = (level − 20) / rate`.
- **Anomaly detection** — a baseline "normal" rate is learned once the window is full, then the 3-sample rate is compared against it.
- **Dashboard estimate** — scikit-learn `LinearRegression` on the last 30 readings predicts the next level.

---

## 🔭 Future work

- Multi-bed support with a central nurse station
- Real SMS / push notifications on escalation
- Real load-cell or optical drip sensor instead of a potentiometer
- Cloud logging and historical analytics

---

## 📄 License

Add your license here (e.g. MIT).

## 👥 Authors

Add your team members' names here.
