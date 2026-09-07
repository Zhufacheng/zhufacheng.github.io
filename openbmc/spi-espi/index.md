# SPI / eSPI

SPI 是一般用途的串列週邊匯流排，BMC 上主要用來掛 SPI NOR flash（firmware 存放處）與 SPI EEPROM 等裝置。eSPI（Enhanced SPI）是取代 LPC 的 host↔BMC 連結，用四個 channel（Peripheral、Virtual Wire、Out-of-Band、Flash Access）分別載入 legacy IO 存取、sideband 訊號、帶外訊息與 host flash 存取。

## 背景概念

- **SPI**：SCLK/MOSI/MISO/CS 四線；BMC SoC 通常有數個 SPI controller channel。
- **SPI NOR flash**：BMC firmware 的存放處；kernel `spi-nor` driver 透過 MTD subsystem 暴露成 `/dev/mtdN`，依 layout 分 partition。
- **`spidev`**：沒有 kernel driver 的 SPI 裝置，用字元裝置 `/dev/spidevX.Y` 讓 userspace raw 讀寫。
- **eSPI**：Intel 規格，host PCH 與 BMC 之間的 point-to-point 連結，取代傳統 LPC。四個 channel：
  - **Peripheral (PVD)**：virtual device 空間，對 host PCH 的 IO/CFG/MMIO 存取（legacy LPC IO/CFG 的繼任）。
  - **Virtual Wire (VW)**：8 條雙向 sideband 訊號——power button、reset 請求、PCH→BMC 中斷、I2C alert 等。
  - **Out-of-Band (OOB)**：雙向 OOB 訊息；載 SMBus-over-OOB 與 power management (PM) 訊息。
  - **Flash Access (FA)**：存取系統 SPI flash（BIOS）——用於 flash recovery／firmware update，也允許 PCH 存取 BMC 端 flash。

## 怎麼運作

### SPI：flash 與 SPI 裝置

1. device tree 描述 SPI controller + NOR flash → `spi-nor` 註冊 MTD → `/dev/mtdN`（依 partition）。
2. BMC firmware layout：U-Boot、kernel、rootfs 各佔 partition；firmware update ＝ 把新 image 寫進對應 MTD partition 並設 boot flag。
3. 其他 SPI 裝置（EEPROM、MCU...）：有 kernel driver 走 sysfs；沒有 driver 就走 `/dev/spidevX.Y` raw 存取。

```
SPI NOR (BMC firmware)
   │ SPI
   ▼
SPI controller (BMC SoC) → spi-nor → MTD (/dev/mtd0..N)
                                │
                                ├─ U-Boot / kernel / rootfs partitions
                                │        ▲
                                │        └─ firmware update: 寫 partition + 設 boot flag
                                │
其他 SPI 裝置 (EEPROM / MCU) → /dev/spidevX.Y → userspace raw 讀寫
```

### eSPI：host 連結

1. **link 建立**：BMC SoC 的 eSPI controller 與 PCH 的 eSPI 構成 point-to-point 連結（mainline kernel 有 NXP 的 `spi-fsl-espi`；Aspeed SoC 的 eSPI driver 由各家 BSP kernel 提供）。
2. **各 channel 的用途**：
   - Peripheral：BMC 存取 PCH virtual device——POST code、PCH 寄存器查詢等。
   - Virtual Wire：事件訊號——power button、reset button、PCH 中斷（BCI）、alert。
   - OOB：**SMBus-over-OOB**——host 端的 IPMI（kernel `ipmi-ssif`）用 OOB 載 SMBus 訊息，不需要實體 SMBus 線；PM 訊息也走這。
   - Flash Access：BMC 讀寫 host 的 **BIOS SPI flash**——host 無法開機時的 flash recovery、或經 BMC 做 firmware update（視平台設計，有走 PCH 協定的 FA channel，也有切實體 mux 直接驅動 flash）。

```
host PCH ────────── eSPI (4 channels) ────────── BMC
                │
  Peripheral:   BMC → PCH 虛擬設備 (IO/CFG/MMIO)
  Virtual Wire: 雙向 sideband (button / reset / alert)
  OOB:          SMBus-over-OOB (IPMI) / PM 訊息
  Flash Access: 存取 BIOS SPI flash (recovery / update)
```

3. **flash recovery 典型流程**：host 開不起來（S5 / FV 失敗）→ BMC 接手 BIOS flash → 寫入已知好的 image → host 重啟。

## 相關專案 / daemon

- kernel：`spi-nor` + MTD、`spidev`、eSPI controller driver（mainline `spi-fsl-espi`；Aspeed 在 vendor BSP kernel）
- `mtd-utils` / `flashcp` — MTD partition 讀寫（firmware update、recovery）
- firmware update daemon（Redfish／`phosphor-software-manager` 一類）— BMC 自己的 update 走 MTD
- IPMI／Redfish 相關 daemon — host 指令通常走 KCS/SSIF（見 Host interface 頁）；SMBus-over-OOB 讓 IPMI 不需要實體 SMBus

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/dev/mtdN` | MTD 裝置，NOR flash partition（firmware 讀寫） |
| `/sys/class/mtd/` | MTD partition 資訊 |
| `/dev/spidevX.Y` | raw SPI 裝置（X = bus、Y = CS） |

## 新手常問

- **SPI 跟 eSPI 的差別？** SPI 是一般四線匯流排（BMC↔週邊）；eSPI 是建立在 SPI 物理層之上的 host↔BMC 協定，多了四個 channel 與 OOB 訊息機制，取代 LPC。
- **LPC 還在嗎？** 老平台有；新 server 幾乎全是 eSPI。eSPI 的 Peripheral channel 相容 legacy LPC 的 IO/CFG 語意，host 端 driver 照舊可用。
- **BMC 做 flash recovery 一定走 eSPI 嗎？** 不一定：有些平台用 GPIO 切實體 SPI mux 直接驅動 BIOS flash，eSPI Flash Access channel 只是其中一條路徑。
- **eSPI 怎麼除錯？** 看 vendor kernel 的 dmesg 與 eSPI controller 暫存器確認 link 狀態（link up、時鐘頻率）；各家 SoC 的除錯介面不同。
