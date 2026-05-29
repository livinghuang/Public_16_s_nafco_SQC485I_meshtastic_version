**繁體中文** | [English](README.en.md)

# SQC485I v2（Meshtastic 版）韌體燒錄與維修說明 / Firmware Flashing Guide

NAFCO RS485-to-Meshtastic 節點（型號 **SQC485I v2**，Meshtastic 版）的官方韌體下載與自助燒錄說明。
當裝置韌體損壞或需要回復出廠韌體時，可依本文件自行重新燒錄。

> Official firmware download & self-repair guide for the NAFCO **SQC485I v2** RS485-to-Meshtastic node.
> This repo only publishes firmware binaries — the source code is kept private.

- **主控晶片 / MCU:** ESP32-C3 + SX1262（LoRa, AS923-TW）
- **韌體版本 / Firmware version:** 請見 [**Releases**](../../releases) 頁面
- **下載 / Download:** 到 [**Releases**](../../releases) 下載**對應節點角色**的韌體包。

---

## 這是什麼 / What this is

SQC485I v2 從 PLC（SPM-8，Modbus RTU slave）經 RS485 讀取資料，透過 Meshtastic LoRa mesh 轉送到後端。
依節點角色分成**兩種韌體**：

| 節點角色 / Role | Meshtastic role | 用途 | 韌體檔名開頭 / Firmware prefix |
|---|---|---|---|
| 現場節點 Field node | `CLIENT` | RS485 讀 PLC，每 15s 上送 payload（portnum 256） | `firmware-sqc485iv2-field-…` |
| 閘道節點 Gateway node | `CLIENT_MUTE` | 收 mesh 封包，WiFi + MQTT 上行到 LAN | `firmware-sqc485iv2-gateway-…` |

完整資料流：

```
[PLC SPM-8 / Modbus RTU slave]
  → RS485 19200 8N1
[SQC485I v2 Field node ×N]    Meshtastic role=CLIENT
  → LoRa AS923 mesh
[SQC485I v2 Gateway node ×1]  Meshtastic role=CLIENT_MUTE
  → WiFi / MQTT
[Mosquitto] → [Decoder] → [InfluxDB] → [Grafana]
```

> ⚠️ 燒錄前請先確認你要燒的是 **field** 還是 **gateway** 韌體，**兩者不可互換**。

---

## 1. 準備工具 / What you need

1. **USB 傳輸線**（直接接 SQC485I 的 USB 埠，ESP32-C3 原生 USB）。
2. **Python 3 + esptool**：先裝 Python 3，再執行 `pip install esptool`。
3. 從 [Releases](../../releases) 下載的**韌體包**。每包內含（請保持所有檔案在**同一資料夾**，安裝腳本會自動尋找配套檔案）：
   - `firmware-sqc485iv2-<role>-<ver>.factory.bin` — 完整出廠映像（**初次燒錄**用）
   - `firmware-sqc485iv2-<role>-<ver>.mt.json` — 分區 metadata（device-install 必要）
   - `mt-esp32c3-ota.bin` — OTA 分區 stub
   - `littlefs-sqc485iv2-<ver>.bin` — 檔案系統分區
   - `firmware-sqc485iv2-<role>-<ver>.bin` — 應用程式映像（**更新**用，保留設定）
   - `device-install.sh` / `device-update.sh`（Meshtastic 官方腳本；Windows 為 `.bat`）

---

## 2. 找出序列埠 / Find the serial port

| 系統 | 埠名稱範例 | 如何查 |
|------|-----------|--------|
| Windows | `COM5` | 裝置管理員 → 連接埠 (COM & LPT) |
| macOS | `/dev/cu.usbmodemXXXX` | 終端機執行 `ls /dev/cu.*` |
| Linux | `/dev/ttyACM0` | 終端機執行 `ls /dev/ttyACM*` |

---

## 3. 燒錄 / Flash

### A. 初次燒錄 — 新裝置或回復出廠（會清除設定）/ Initial flash (erases settings)

解壓韌體包，在該資料夾內開啟終端機，把 `<PORT>` 換成上一步查到的埠、`<role>` 換成 `field` 或 `gateway`：

**macOS / Linux**

```bash
./device-install.sh -p <PORT> -f firmware-sqc485iv2-<role>-<ver>.factory.bin
```

**Windows**

```bat
device-install.bat -p COM5 -f firmware-sqc485iv2-<role>-<ver>.factory.bin
```

不想用腳本，也可純手動燒完整出廠映像：

```bash
esptool.py --chip esp32c3 --port <PORT> --baud 921600 write_flash 0x0 firmware-sqc485iv2-<role>-<ver>.factory.bin
```

看到 `Hash of data verified.` 並 `Hard resetting...` 即代表成功。

### B. 更新韌體 — 保留設定（日常維修建議用這個）/ Update (keeps settings)

```bash
./device-update.sh -p <PORT> -f firmware-sqc485iv2-<role>-<ver>.bin
```

> 更新用的是 `.bin`（**不是** `.factory.bin`），只更新應用程式分區，節點設定（金鑰、channel、role）會保留。

---

## 4. ⚠️ 重要：初次燒錄會清除節點身分 / Initial flash wipes node identity

- Meshtastic 節點的**身分金鑰（PKI 公私鑰）、channel 設定（NAFCO channel）、region（AS923-TW）、RS485／角色設定**都存在 flash 的 NVS。
- **初次燒錄／出廠映像會清掉這些** → 節點會產生**新身分**，需重新設定 channel、region、role，並重新加入 mesh。
- **日常維修／更新請改用「B. 更新韌體」（`device-update.sh` + 純 `.bin`）**，保留設定，無需重新註冊。

> Initial/factory flash wipes the node's PKI keys, channel, region and role. For normal repair use the
> update path (plain `.bin`) so settings are preserved and no re-provisioning is needed.

---

## 5. 韌體完整性 / Firmware integrity (SHA-256)

每個 Release 的說明會附上各檔案的 SHA-256。下載後可比對，確認檔案未損毀：

```bash
shasum -a 256 <file>          # macOS / Linux
certutil -hashfile <file> SHA256   # Windows
```

---

## 6. 疑難排解 / Troubleshooting

| 症狀 | 處理方式 |
|------|----------|
| 找不到序列埠 / 連線失敗 | 換條 USB 線、確認驅動已安裝；關閉佔用該埠的程式（如序列監看視窗）。 |
| `Failed to connect` / 無法進入下載模式 | 按住裝置上的 **BOOT** 鍵不放，插上 USB（或按一下 **RESET**），再放開 BOOT，重跑指令。 |
| 燒完不上線 | 確認燒的是**對的角色**（field / gateway）；初次燒錄後需重新設定 channel / region / role。 |
| 燒完設定不見了 | 你用了初次燒錄（`.factory.bin`）。日常更新請改用 `device-update.sh` + 純 `.bin`。 |

---

## 支援 / Support

韌體與硬體相關問題請聯絡供應商。本頁僅提供出廠韌體下載與燒錄說明，**不含原始碼，亦不收 issue**。
