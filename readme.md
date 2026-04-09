# Rail Track Geometry Inspection System — BBB Industrial Setup

> **Target:** BeagleBone Black Industrial · Debian Trixie (2026)  
> **Display:** 1024×600 HDMI/Touch · PyQt5 GUI  
> **Sensors:** Rotary Encoder (GPIO), Gauge/Cross-Level Pots (ADC), u-blox NEO-M8P-2 (UART4), EC200U LTE (eth1)

---

## Hardware Wiring

| Sensor             | BBB Pin | Function      | Wire To             |
|--------------------|---------|---------------|----------------------|
| Gauge Pot (Pin 1)  | P9.32   | VDD_ADC 1.8V  | Pot VCC (Pin 1)      |
| Gauge Pot (Pin 3)  | P9.34   | GNDA_ADC      | Pot GND (Pin 3)      |
| Gauge Pot (Wiper)  | P9.39   | AIN0          | Pot Wiper (Pin 2)    |
| Cross Pot (Pin 1)  | P9.32   | VDD_ADC 1.8V  | Pot VCC (Pin 1)      |
| Cross Pot (Pin 3)  | P9.34   | GNDA_ADC      | Pot GND (Pin 3)      |
| Cross Pot (Wiper)  | P9.40   | AIN1          | Pot Wiper (Pin 2)    |
| Encoder CLK        | P8.11   | GPIO1_13 (525)| Encoder CLK          |
| Encoder DT         | P8.12   | GPIO1_12 (524)| Encoder DT           |
| Encoder SW         | P8.14   | GPIO0_26 (634)| Encoder Switch       |
| GPS TX             | P9.11   | UART4_RX      | u-blox TX            |
| GPS / Encoder VCC  | P9.3    | 3.3V          | Sensor VCC           |
| GPS / Encoder GND  | P9.1    | DGND          | Sensor GND           |

> **⚠️ CRITICAL:** Potentiometers MUST use **1.8V (P9.32)** — NOT 3.3V. Connecting 3.3V to AIN pins will permanently damage the AM335x processor.

---

## Fresh BBB Setup — Copy & Paste All Commands

SSH into your BBB as the `debian` user, then copy-paste the **entire block below** into the terminal and press Enter:

```bash
# ─── 1. UPDATE SYSTEM ────────────────────────────────────────────────────────
sudo apt update && \
sudo apt upgrade -y && \

# ─── 2. INSTALL ALL DEPENDENCIES ─────────────────────────────────────────────
sudo apt install -y \
  python3 \
  python3-pip \
  python3-pyqt5 \
  python3-serial \
  cpufrequtils \
  git \
  libxcb-xinerama0 \
  x11-utils \
  ca-certificates && \

# ─── 3. ADD USER TO HARDWARE GROUPS ──────────────────────────────────────────
sudo usermod -a -G gpio debian && \
sudo usermod -a -G dialout debian && \
sudo usermod -a -G iio debian && \

# ─── 4. CREATE APPLICATION DIRECTORIES ───────────────────────────────────────
mkdir -p /home/debian/trolley && \
mkdir -p /home/debian/surveys && \

# ─── 5. COPY APPLICATION FILES ───────────────────────────────────────────────
# (If you have the files on a USB stick mounted at /media/debian/USB, adjust
#  the source path below. Otherwise, copy them manually before this step.)
# cp /media/debian/USB/testing/* /home/debian/trolley/ && \

# ─── 6. ENABLE UART4 OVERLAY (GPS on /dev/ttyS4) ─────────────────────────────
sudo grep -q "BB-UART4" /boot/uEnv.txt || \
  echo "uboot_overlay_addr4=/lib/firmware/BB-UART4-00A0.dtbo" | sudo tee -a /boot/uEnv.txt && \

# ─── 7. ENABLE ADC OVERLAY (Potentiometers) ──────────────────────────────────
sudo grep -q "BB-ADC" /boot/uEnv.txt || \
  echo "uboot_overlay_addr5=/lib/firmware/BB-ADC-00A0.dtbo" | sudo tee -a /boot/uEnv.txt && \

# ─── 8. LOAD ADC KERNEL MODULE NOW ──────────────────────────────────────────
sudo modprobe ti_am335x_adc && \

# ─── 9. CREATE DEFAULT CONFIG ────────────────────────────────────────────────
cat > /home/debian/trolley/rail_config.json << 'CONFIGEOF'
{
  "csv_dir": "/home/debian/surveys",
  "hl_sec": 30,
  "server": "8.8.8.8",
  "lte_iface": "eth1",
  "encoder": {
    "scale": 1.0,
    "ppr": 20,
    "diam": 62.0,
    "calibrated": false
  },
  "adc": {
    "zero": 2048,
    "mpc": 0.0684,
    "calibrated": false
  },
  "incl": {
    "offset": 0.0,
    "calibrated": false
  },
  "gnss": {
    "ref_ch": 0.0,
    "calibrated": false,
    "origin_lat": 28.6139,
    "origin_lon": 77.2090
  }
}
CONFIGEOF

# ─── 10. INSTALL SYSTEMD SERVICE ─────────────────────────────────────────────
sudo cp /home/debian/trolley/trolley.service /etc/systemd/system/trolley.service && \
sudo systemctl daemon-reload && \
sudo systemctl enable trolley.service && \

# ─── 11. SET CPU GOVERNOR TO PERFORMANCE ─────────────────────────────────────
sudo cpufreq-set -g performance && \

# ─── 12. DONE — REBOOT ───────────────────────────────────────────────────────
echo "" && \
echo "=============================================" && \
echo "  SETUP COMPLETE — REBOOTING IN 5 SECONDS"    && \
echo "=============================================" && \
echo "" && \
sleep 5 && \
sudo reboot
```

---

## After Reboot — Verify Hardware

SSH back in and run the sensor test:

```bash
cd /home/debian/trolley
python3 bbb_sensor_test.py
```

You should see:
- **ADC0 (Gauge):** `FOUND` — raw value 0–4095 changing as you turn the pot
- **ADC1 (Cross):** `FOUND` — raw value 0–4095 changing as you turn the pot
- **Encoder:** `OK` — count incrementing as you rotate the wheel
- **GPS:** `FOUND` → `FIX` after 30–90 seconds outdoors

Press `Ctrl+C` to see the full diagnostic report.

---

## Check the Systemd Service

```bash
sudo systemctl status trolley.service
```

The GUI should auto-start on boot with the HDMI display connected.

### Manual GUI Launch (for debugging)

```bash
DISPLAY=:0 python3 /home/debian/trolley/integrated_rail.py
```

---

## Troubleshooting

### ADC Not Found
```bash
sudo modprobe ti_am335x_adc
ls /sys/bus/iio/devices/iio:device0/
```
If the directory doesn't exist, add the ADC overlay to `/boot/uEnv.txt`:
```
uboot_overlay_addr5=/lib/firmware/BB-ADC-00A0.dtbo
```
Then reboot.

### GPS Port Missing (/dev/ttyS4)
```bash
ls /dev/ttyS4
```
If missing, verify the UART4 overlay is in `/boot/uEnv.txt`:
```
uboot_overlay_addr4=/lib/firmware/BB-UART4-00A0.dtbo
```
Check available overlays:
```bash
ls /lib/firmware/ | grep UART
```
Then reboot.

### GUI Won't Start
```bash
# Check X server is running
echo $DISPLAY
# Should output :0

# Check Xauthority
ls -la /home/debian/.Xauthority

# Try launching with explicit environment
export DISPLAY=:0
export XDG_RUNTIME_DIR=/tmp/runtime-debian
mkdir -p /tmp/runtime-debian
chmod 700 /tmp/runtime-debian
python3 /home/debian/trolley/integrated_rail.py
```

### Encoder Not Counting
Verify GPIO export:
```bash
cat /sys/class/gpio/gpio525/value
cat /sys/class/gpio/gpio524/value
cat /sys/class/gpio/gpio634/value
```
If files don't exist:
```bash
echo 525 | sudo tee /sys/class/gpio/export
echo 524 | sudo tee /sys/class/gpio/export
echo 634 | sudo tee /sys/class/gpio/export
```

---

## File Structure on BBB

```
/home/debian/trolley/
├── integrated_rail.py      # Main GUI application
├── bbb_sensor_test.py      # Terminal-based sensor diagnostic tool
├── trolley.service          # Systemd unit file
└── rail_config.json         # Runtime configuration (auto-created)

/home/debian/surveys/        # CSV data output directory
```

---

## Notes

- **OS:** Debian Trixie (2026-03-17 image). `config-pin` is **removed** — all pin muxing is done via `/boot/uEnv.txt` overlays at boot.
- **Python:** System Python 3 with `python3-pyqt5` and `python3-serial` from apt (no virtualenv needed).
- **Service:** Runs as `debian` user with `nice -n -10` for real-time priority. Restarts automatically on crash.
- **Display:** 1024×600 HDMI. The GUI is designed for this exact resolution.
