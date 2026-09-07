# OpenBMC community 的 features

OpenBMC community 裡比較大的 feature，依領域分類，每項附主要對應的專案/模組。

## 管理協議（Management Protocols）

- **IPMI** — IPMI 1.5 / 2.0。LAN 端用 RMCP+（UDP 623），host 端用 KCS / BT 介面。提供 SDR（sensor 記錄）、SEL（事件紀錄）、FRU、power control、user mgmt。相關專案：`phosphor-ipmi-host`（KCS/BT/host 端）、`phosphor-ipmi-net`（LAN/RMCP+）。
- **Redfish** — DMTF 標準的 HTTP/HTTPS 管理介面，由 `bmcweb` 實作。提供 Chassis、System、Manager、UpdateService、LogService、TaskService 等；內部做 D-Bus → Redfish 的 mapping。
- **MCTP** — Management Component Transport Protocol（DSP0236），承載 PLDM 等管理訊息的傳輸層；可跑在 I2C/SMBus、PCIe VDM、KCS 上。
- **PLDM** — Platform Data Model Message（DSP0240），跑在 MCTP 上的訊息類型：BIOS messages、SMBIOS、Firmware Update（FSP）、Sensor、Base/OEM、FRU。
- **NCSI / PNCC** — 網路控制器側帶介面，讓 BMC 能 out-of-band 讀寫網路控制器（NCSI 指令、mux/PHY 狀態）。

## 遙測與感測（Sensors & Telemetry）

- **`phosphor-hwmon-sensor`** — 把 Linux hwmon 裝置（溫度、電壓等）變成 D-Bus sensor。
- **`phosphor-reading-monitor`** — 把 sensor 讀值跟 threshold 比對，超標就丟事件。
- **`phosphor-virtual-sensor`** — 由多個真實 sensor 算出虛擬/合成值（sum、max、min、avg、ratio 等）。
- **ADC sensor**（`phosphor-adc-sensor`）— 讀 board 上的 ADC 值。
- **Power telemetry** — 讀功率/能耗值。

## 電源與熱管理（Power & Thermal）

- **`phosphor-power`** — 電源狀態管理（on/off/cycle/reset）、power sequencing、power fault 處理。
- **`phosphor-pid-control`** — 風扇轉速控制：讀溫度 → PID 算目標 PWM → 寫回風扇（D-Bus `FanPwm.Target`）。見 [phosphor-pid-control](../phosphor-pid-control/)。
- **Power capping** — 限制 host/元件的功率上限。
- **PSU 管理**（`phosphor-psu-manager`）— 監控電源供應器狀態、冗餘、功率。

## 韌體更新（Firmware Update）

- **`phosphor-software-manager`** — BMC image 更新：版本管理、activation、A/B 雙映像、rollback / failsafe。
- **Host firmware update** — 用 PLDM FSP 或 capsule 更新 host 端韌體（BIOS、CPLD、NVMe 等）。
- **U-Boot / kernel / rootfs** — BMC 自身的 boot chain。

## 紀錄與事件（Logging & Events）

- **`phosphor-logging`** — 統一事件記錄，可對應到 IPMI SEL。
- **SEL（System Event Log）** — IPMI 事件紀錄（時間戳、sensor、event）。
- **D-Bus event logging** — 訂閱 D-Bus signal 記錄事件。
- **crashdump / coredump** — out-of-band 收集 host 的 crashdump / coredump。

## 設備清單與發現（Inventory & Discovery）

- **`entity-manager`** — 硬體 inventory / 配置管理：解析 FRU/EC/JTOD 的 presence 與配置（JSON），產出 D-Bus 物件（含 PID 設定）。見 [entity-manager](../entity-manager/)。
- **`phosphor-inventory`** — 硬體 inventory。
- **FRU（Field Replaceable Unit）** — 可更換單元的資訊（EEPROM）。

## 網路與時間（Network & Time）

- **網路管理**（`phosphor-network`）— DHCP / static IP、VLAN、NTP、DNS、zero-config。
- **Time** — NTP 同步、時間管理。

## 安全與存取（Security & Access）

- **`phosphor-user-manager`** — 使用者帳號管理（帳號、密碼、權限層級）。
- **KVM over IP** — 遠端桌面。
- **SOL（Serial-over-LAN）** — 串列埠過網路。
- **Secure boot** — BMC 安全啟動。
- **Attestation** — BMC / host 驗證。

## 硬體介面（Hardware Interfaces）

- **I2C / I3C** — sensor、裝置匯流排。
- **SPI / eSPI** — 含 OOB、flash recovery。
- **Host interface** — KCS / BT / LPC-eSPI / serial。
- **Video / USB** — 影片、USB 裝置模擬。
- **Watchdog** — 看門狗計時器。

## 基礎架構（Infrastructure）

- **D-Bus**（`sdbusplus`）— 核心 IPC。
- **systemd** — 服務管理。
- **Yocto / bitbake** — build system。
