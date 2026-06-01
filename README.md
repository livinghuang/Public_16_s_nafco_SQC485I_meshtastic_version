**繁體中文** | [English](README.en.md)

# SQC485I v2（Meshtastic 版）韌體燒錄與維修說明 / Firmware Flashing Guide

NAFCO RS485-to-Meshtastic 場域節點（型號 **SQC485I v2**，**Meshtastic 網狀網路版**）的官方韌體下載與自助燒錄說明。當裝置韌體損壞或需要回復出廠韌體時，可依本文件自行重新燒錄。

> Official firmware download & self-repair guide for the NAFCO **SQC485I v2** RS485-to-Meshtastic field node.

- **韌體版本 / Firmware version:** `v1.0.1`（基於 Meshtastic `2.7.20`；NAFCO 場域節點客製，RS485 Modbus → Meshtastic mesh 轉發）
- **主控晶片 / MCU:** ESP32-C3 + SX1262（LoRa）
- **無線區域 / Region:** AS923-TW（台灣）

> 本機是 **Meshtastic 網狀網路節點**，不是 LoRaWAN 裝置。它從 RS485 讀取 PLC（Modbus RTU，FC03）資料，透過 Meshtastic LoRa mesh 轉發；mesh 由 linxdot 上的 USB 接收器收斂進 MQTT。

---

## 0. ⚠️ 先確認你的板子版本（v231 或 v233）/ Pick the right firmware first

本產品有兩個硬體版本，**RS485 的 DE（傳送致能）腳位極性相反**。燒錯版本會讓 RS485 收發方向顛倒，**Modbus 完全讀不到資料**（裝置看似正常開機、也會上 mesh，但永遠回報不到 PLC 數值）。請務必依下表選對韌體：

| 板子版本 | 如何辨識（外觀） | 對應韌體資料夾 |
|---|---|---|
| **v233**（較新，2026-05 之後） | 對外是**單一 8-pin 連接器 `CN1`** + 獨立 **2-pin 電池座 `CN2`**；板上**有**內建 2.4G 晶片天線 | [`firmware/v233/`](firmware/v233/) |
| **v231**（較舊 / Lot 1，2026-02） | 對外是 **4-pin `CONN2` + 6-pin `CONN3`** 兩個連接器；板上**沒有**內建 2.4G 天線 | [`firmware/v231/`](firmware/v231/) |

> **技術原因**：v231 用單路反相器（U6），RS485 的 `DE` 由 `GPIO9` **直接驅動**（HIGH = 傳送）；v233 改用雙路反相器（U7），`GPIO9` 經反相後才到 `DE`（LOW = 傳送）。兩版唯一影響韌體的差異就是這個極性，因此分成兩包。
>
> 不確定版本時，**先看對外連接器是 8-pin 單顆（v233）還是 4-pin+6-pin 兩顆（v231）** 最快。

下載 / Download：請至 [**Releases**](../../releases) 頁面，或直接取用對應 `firmware/v2xx/` 資料夾內的 `firmware-merged.bin`（單一合併檔，已含 bootloader / 分割表 / 應用程式）。**以下指令把 `<變體>` 換成你的 `v233` 或 `v231`。**

---

## 1. 準備工具 / What you need

1. **USB 傳輸線**（直接接 SQC485I 的 USB 埠，ESP32-C3 原生 USB）。
2. **esptool**（Espressif 官方燒錄工具，跨 Windows / macOS / Linux）。先安裝 Python 3，再執行：

   ```bash
   pip install esptool
   ```

3. 對應版本的韌體檔 **`firmware-merged.bin`**（在 `firmware/v233/` 或 `firmware/v231/` 內）。

---

## 2. 找出序列埠 / Find the serial port

| 系統 | 埠名稱範例 | 如何查 |
|------|-----------|--------|
| Windows | `COM5` | 裝置管理員 → 連接埠 (COM & LPT) |
| macOS | `/dev/cu.usbmodemXXXX` | 終端機執行 `ls /dev/cu.*` |
| Linux | `/dev/ttyACM0` | 終端機執行 `ls /dev/ttyACM*` |

---

## 3. 燒錄韌體（單一檔案，一行指令）/ Flash

在放有對應 `firmware-merged.bin` 的資料夾內開啟終端機，把 `<PORT>` 換成上一步查到的埠：

**macOS / Linux**

```bash
esptool.py --chip esp32c3 --port <PORT> --baud 921600 write_flash 0x0 firmware-merged.bin
```

**Windows（換成 COM 埠）**

```bat
esptool.py --chip esp32c3 --port COM5 --baud 921600 write_flash 0x0 firmware-merged.bin
```

看到 `Hash of data verified.` 並 `Hard resetting...` 即代表成功。裝置會自動重開機並恢復正常運作。

---

## 4. ⚠️ 重要：請勿執行 erase / Do NOT erase

- **正常維修請「不要」加上 `erase_flash` 或 `--erase`**。
- 裝置的 **Meshtastic 設定**（無線區域 TW、channel 與 PSK 金鑰、節點角色 role、RS485/Serial 模組設定）存放在 NVS。一旦清除，這些設定會全部歸零，裝置將**無法自動加入 NAFCO mesh**，必須重新做一次出廠設定（重設 region / channel / PSK / role）才能恢復轉發 RS485 資料。
- 節點編號（node ID）由晶片 MAC 推導，**重新燒錄不會改變**；但 channel PSK 與模組設定**不會**自動帶回——清除後必須重設。
- 只用上面的 `write_flash`（不加 erase）重燒，原本的設定會保留，裝置會自動重新加入 mesh。

---

## 5. 韌體完整性 / Firmware integrity (SHA-256)

下載後可比對雜湊值，確認檔案未損毀：

| 版本 / Variant | 檔案 / File | SHA-256 |
|------|------|---------|
| **v233** | `firmware/v233/firmware-merged.bin` | `4e6078ebf5be5294d4fda91c3e7ab0748921007efa7e41742455ec1256fef379` |
| **v231** | `firmware/v231/firmware-merged.bin` | `9d8e6430e5c9c3f9e52b3a176156c142d26b02235dd8ca78c1f347b7325ccfad` |

驗證指令：`shasum -a 256 firmware-merged.bin`（macOS/Linux）或 `certutil -hashfile firmware-merged.bin SHA256`（Windows）。

---

## 6. 疑難排解 / Troubleshooting

| 症狀 | 處理方式 |
|------|----------|
| 找不到序列埠 / 連線失敗 | 換條 USB 線、確認驅動已安裝；關閉佔用該埠的程式（如序列監看視窗、Meshtastic App）。 |
| `Failed to connect` / 無法進入下載模式 | 按住裝置上的 **BOOT** 鍵不放，插上 USB（或按一下 **RESET**），再放開 BOOT，重跑指令。 |
| 燒完仍不運作 | 確認檔名與埠正確；再執行一次燒錄。 |
| **裝置有上 mesh、但 PLC 數值永遠讀不到** | **十之八九是燒錯板子版本（v231/v233 DE 極性相反）**。回到第 0 節，依連接器外觀重新確認版本，改燒正確的韌體。 |
| 燒完後沒有資料上傳 mesh | 確認設定沒被清掉（region TW / channel / PSK / role），並檢查 RS485 接線與 PLC（Modbus slave）。 |

---

## 進階 / Advanced

若需個別燒錄分段檔，各 `firmware/v2xx/` 內仍提供 `bootloader.bin`（`0x0`）、
`partitions.bin`（`0x8000`）、`boot_app0.bin`（`0xe000`）、`firmware.bin`（`0x10000`）。
一般客戶用對應的 `firmware-merged.bin` 即可，無需理會這些。

---

## 支援 / Support

韌體與硬體相關問題請聯絡供應商。本頁僅提供出廠韌體下載與燒錄說明，不含原始碼。
