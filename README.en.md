[繁體中文](README.md) | **English**

# SQC485I v2 (Meshtastic) Firmware Flashing & Repair Guide

Official firmware download and self-flashing guide for the NAFCO RS485-to-Meshtastic field node
(model **SQC485I v2**, **Meshtastic mesh version**). If a unit's firmware becomes corrupted or you
need to restore the factory firmware, you can re-flash it yourself by following this guide.

- **Firmware version:** `v1.0.1` (based on Meshtastic `2.7.20`; NAFCO field-node customization, RS485 Modbus → Meshtastic mesh forwarding)
- **MCU:** ESP32-C3 + SX1262 (LoRa)
- **Region:** AS923-TW (Taiwan)

> This unit is a **Meshtastic mesh node**, not a LoRaWAN device. It reads PLC data over RS485
> (Modbus RTU, FC03) and forwards it across a Meshtastic LoRa mesh; the mesh is collected into MQTT
> by the USB receiver on the linxdot box.

---

## 0. ⚠️ Pick the right firmware first (v231 or v233)

This product ships in two hardware revisions whose **RS485 DE (transmit-enable) pin polarity is
reversed**. Flashing the wrong one makes RS485 transmit/receive run backwards — **Modbus reads
nothing** (the unit boots fine and even joins the mesh, but never reports any PLC values). Use the
table below to choose:

| Board rev | How to identify (by sight) | Firmware folder |
|---|---|---|
| **v233** (newer, after 2026-05) | A single **8-pin connector `CN1`** + a separate **2-pin battery connector `CN2`**; an onboard 2.4 GHz chip antenna **is present** | [`firmware/v233/`](firmware/v233/) |
| **v231** (older / Lot 1, 2026-02) | Two connectors: **4-pin `CONN2` + 6-pin `CONN3`**; **no** onboard 2.4 GHz antenna | [`firmware/v231/`](firmware/v231/) |

> **Why:** v231 uses a single inverter (U6) and drives RS485 `DE` **directly from `GPIO9`**
> (HIGH = transmit); v233 uses a dual inverter (U7) so `GPIO9` is inverted before `DE`
> (LOW = transmit). This polarity is the only firmware-visible difference, hence two builds.
>
> If unsure, the fastest tell is the external connector: **one 8-pin (v233) vs. a 4-pin + 6-pin
> pair (v231)**.

**Download:** from the [**Releases**](../../releases) page, or take `firmware-merged.bin` directly
from the matching `firmware/v2xx/` folder (a single merged image containing bootloader, partition
table, and application). **In the commands below, use the `firmware-merged.bin` from your variant.**

---

## 1. What you need

1. A **USB cable** (connect directly to the SQC485I USB port — ESP32-C3 native USB).
2. **esptool** (Espressif's official flashing tool, cross-platform Windows / macOS / Linux).
   Install Python 3, then run:

   ```bash
   pip install esptool
   ```

3. The matching **`firmware-merged.bin`** (from `firmware/v233/` or `firmware/v231/`).

---

## 2. Find the serial port

| OS | Example port | How to find it |
|------|-----------|--------|
| Windows | `COM5` | Device Manager → Ports (COM & LPT) |
| macOS | `/dev/cu.usbmodemXXXX` | Run `ls /dev/cu.*` in a terminal |
| Linux | `/dev/ttyACM0` | Run `ls /dev/ttyACM*` in a terminal |

---

## 3. Flash the firmware (single file, one command)

Open a terminal in the folder containing your variant's `firmware-merged.bin`, and replace `<PORT>`
with the port from the previous step:

**macOS / Linux**

```bash
esptool.py --chip esp32c3 --port <PORT> --baud 921600 write_flash 0x0 firmware-merged.bin
```

**Windows (with a COM port)**

```bat
esptool.py --chip esp32c3 --port COM5 --baud 921600 write_flash 0x0 firmware-merged.bin
```

`Hash of data verified.` followed by `Hard resetting...` means success. The device reboots and
resumes normal operation automatically.

---

## 4. ⚠️ Important: do NOT erase

- **For a normal repair, do NOT add `erase_flash` or `--erase`.**
- The device's **Meshtastic configuration** (region TW, channel + PSK key, device role, and the
  RS485/Serial module settings) is stored in NVS. Erasing it resets all of these, and the node will
  **fail to rejoin the NAFCO mesh** until it is fully re-provisioned (region / channel / PSK / role
  set again) before it can forward RS485 data.
- The node ID is derived from the chip's MAC and **survives a reflash**; but the channel PSK and
  module settings do **not** — they must be reapplied after an erase.
- A plain `write_flash` (no erase) keeps the existing config, so the device rejoins the mesh
  automatically.

---

## 5. Firmware integrity (SHA-256)

After downloading, verify the hash to confirm the file is intact:

| Variant | File | SHA-256 |
|------|------|---------|
| **v233** | `firmware/v233/firmware-merged.bin` | `4e6078ebf5be5294d4fda91c3e7ab0748921007efa7e41742455ec1256fef379` |
| **v231** | `firmware/v231/firmware-merged.bin` | `9d8e6430e5c9c3f9e52b3a176156c142d26b02235dd8ca78c1f347b7325ccfad` |

Verify with: `shasum -a 256 firmware-merged.bin` (macOS/Linux) or
`certutil -hashfile firmware-merged.bin SHA256` (Windows).

---

## 6. Troubleshooting

| Symptom | Fix |
|------|----------|
| Port not found / connection failed | Try another USB cable; confirm the driver is installed; close any program holding the port (e.g. a serial monitor or the Meshtastic app). |
| `Failed to connect` / cannot enter download mode | Hold the **BOOT** button, plug in USB (or press **RESET**), then release BOOT and re-run the command. |
| Still not working after flashing | Confirm the filename and port are correct, then flash again. |
| **Unit joins the mesh but PLC values never arrive** | **Almost always the wrong board variant was flashed (v231/v233 DE polarity is reversed).** Go back to Section 0, re-check the board by its connectors, and flash the correct firmware. |
| No data reaching the mesh after flashing | Confirm the config was not erased (region TW / channel / PSK / role), and check the RS485 wiring and PLC (Modbus slave). |

---

## Advanced

If you need to flash the separate segment files instead, each `firmware/v2xx/` folder still
provides `bootloader.bin` (`0x0`), `partitions.bin` (`0x8000`), `boot_app0.bin` (`0xe000`),
and `firmware.bin` (`0x10000`). Most customers should just use the matching `firmware-merged.bin`.

---

## Support

For firmware and hardware questions, please contact your supplier. This page provides factory
firmware download and flashing instructions only; it does not include source code.
