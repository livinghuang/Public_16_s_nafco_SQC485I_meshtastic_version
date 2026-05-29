[繁體中文](README.md) | **English**

# SQC485I v2 (Meshtastic) Firmware Flashing & Repair Guide

Official firmware download & self-repair guide for the NAFCO **SQC485I v2** RS485-to-Meshtastic node.
If the device firmware is corrupted or you need to restore factory firmware, follow this document to reflash it yourself.

> This repo only publishes firmware binaries — the source code is kept private.

- **MCU:** ESP32-C3 + SX1262 (LoRa, AS923-TW)
- **Firmware version:** see the [**Releases**](../../releases) page
- **Download:** grab the firmware package for the **matching node role** from [**Releases**](../../releases).

---

## What this is

The SQC485I v2 reads data from a PLC (SPM-8, Modbus RTU slave) over RS485 and forwards it across a
Meshtastic LoRa mesh to the backend. There are **two firmware builds** by node role:

| Role | Meshtastic role | Purpose | Firmware prefix |
|---|---|---|---|
| Field node | `CLIENT` | Reads the PLC over RS485, uplinks a payload every 15s (portnum 256) | `firmware-sqc485iv2-field-…` |
| Gateway node | `CLIENT_MUTE` | Receives mesh packets, uplinks via WiFi + MQTT to the LAN | `firmware-sqc485iv2-gateway-…` |

Data flow:

```
[PLC SPM-8 / Modbus RTU slave]
  → RS485 19200 8N1
[SQC485I v2 Field node ×N]    Meshtastic role=CLIENT
  → LoRa AS923 mesh
[SQC485I v2 Gateway node ×1]  Meshtastic role=CLIENT_MUTE
  → WiFi / MQTT
[Mosquitto] → [Decoder] → [InfluxDB] → [Grafana]
```

> ⚠️ Before flashing, confirm whether you need the **field** or **gateway** firmware — **they are not interchangeable**.

---

## 1. What you need

1. **A USB cable** (connect directly to the SQC485I USB port — native ESP32-C3 USB).
2. **Python 3 + esptool:** install Python 3, then run `pip install esptool`.
3. The **firmware package** downloaded from [Releases](../../releases). Keep **all files in the same folder**
   (the install script locates the companion files automatically). Each package contains:
   - `firmware-sqc485iv2-<role>-<ver>.factory.bin` — full factory image (**initial flash**)
   - `firmware-sqc485iv2-<role>-<ver>.mt.json` — partition metadata (required by device-install)
   - `mt-esp32c3-ota.bin` — OTA partition stub
   - `littlefs-sqc485iv2-<ver>.bin` — filesystem partition
   - `firmware-sqc485iv2-<role>-<ver>.bin` — application image (**update**, keeps settings)
   - `device-install.sh` / `device-update.sh` (official Meshtastic scripts; `.bat` on Windows)

---

## 2. Find the serial port

| OS | Example port | How to find it |
|------|-----------|--------|
| Windows | `COM5` | Device Manager → Ports (COM & LPT) |
| macOS | `/dev/cu.usbmodemXXXX` | Run `ls /dev/cu.*` in a terminal |
| Linux | `/dev/ttyACM0` | Run `ls /dev/ttyACM*` in a terminal |

---

## 3. Flash

### A. Initial flash — new device or factory restore (erases settings)

Unzip the package, open a terminal in that folder, replace `<PORT>` with the port from step 2 and
`<role>` with `field` or `gateway`:

**macOS / Linux**

```bash
./device-install.sh -p <PORT> -f firmware-sqc485iv2-<role>-<ver>.factory.bin
```

**Windows**

```bat
device-install.bat -p COM5 -f firmware-sqc485iv2-<role>-<ver>.factory.bin
```

Prefer no script? Flash the full factory image manually:

```bash
esptool.py --chip esp32c3 --port <PORT> --baud 921600 write_flash 0x0 firmware-sqc485iv2-<role>-<ver>.factory.bin
```

`Hash of data verified.` and `Hard resetting...` means success.

### B. Update — keeps settings (recommended for normal repair)

```bash
./device-update.sh -p <PORT> -f firmware-sqc485iv2-<role>-<ver>.bin
```

> Update uses the plain `.bin` (**not** `.factory.bin`); it rewrites only the app partition, so the
> node's settings (keys, channel, role) are preserved.

---

## 4. ⚠️ Important: initial flash wipes node identity

- A Meshtastic node's **PKI key pair, channel config (NAFCO channel), region (AS923-TW) and RS485/role
  settings** all live in flash NVS.
- **An initial/factory flash erases them** → the node gets a **new identity** and must be re-configured
  (channel, region, role) and re-added to the mesh.
- **For normal repair/updates use path B (`device-update.sh` + plain `.bin`)** to keep settings — no
  re-provisioning needed.

---

## 5. Firmware integrity (SHA-256)

Each Release lists the SHA-256 of every file. After downloading, verify the file is intact:

```bash
shasum -a 256 <file>               # macOS / Linux
certutil -hashfile <file> SHA256   # Windows
```

---

## 6. Troubleshooting

| Symptom | What to do |
|------|----------|
| Port not found / connection fails | Try another USB cable, confirm drivers are installed; close any program holding the port (e.g. a serial monitor). |
| `Failed to connect` / can't enter download mode | Hold the **BOOT** button, plug in USB (or tap **RESET**), release BOOT, then rerun. |
| Flashed but node never joins | Confirm you flashed the **correct role** (field / gateway); after an initial flash you must reconfigure channel / region / role. |
| Settings disappeared after flashing | You used the initial flash (`.factory.bin`). For updates use `device-update.sh` + plain `.bin`. |

---

## Support

For firmware and hardware issues, contact your supplier. This page only provides factory firmware
download and flashing instructions — **no source code, and issues are not accepted here**.
