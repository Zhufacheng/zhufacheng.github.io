# IPMI

IPMI (Intelligent Platform Management Interface) 是 legacy 管理協定，在 OpenBMC 上仍然是第一級管理介面：提供 host 端（KCS / BT）與 LAN（RMCP+，UDP 623）兩條路徑的 out-of-band 管理。典型命令涵蓋 SDR、SEL、FRU、power control 與 user management。

## 背景概念

- **IP command**：IPMI 訊息以 `NetFn/LUN + Command` 識別（例如 Get Device ID、Read FRU Data、Set System Power State、Get SEL Entry Information）。IPMI 1.5 幾乎沒有驗證；IPMI 2.0 加上 RAKP session 認證（RMCP+）。
- **RMCP+**：IPMI 2.0 的 LAN transport，跑在 UDP port 623。session 經 RAKP 4 步握手建立（cipher suite 選擇、user 認證）。`ipmitool -I lanplus` 是最常見的 client。
- **KCS / BT**：host 與 BMC 之間的 IPMI channel。KCS（Keyboard Controller Style）是 LPC 上 FIFO 式註冊介面；BT（Bit Write）是較舊的 I2C 型協定。host BIOS POST、`ipmitool -I kcs` 這類工具都走這條路。
- **SDR**（Sensor Data Record）：感測器的靜態描述（讀取類型、unit、reading 方式）。讀值走 Get Sensor Reading。
- **SEL**（System Event Log）：power / state / sensor threshold 事件記錄。
- **FRU**：field-replaceable unit 的 EEPROM（i2c-fru 格式），記錄 part number、serial、製造日期等。

## 怎麼運作

兩條 command 路徑，最後都匯流到同一個 command engine：

```
Host（KCS over LPC / BT）
   │  host 寫命令 + data 進 KCS FIFO
   ▼
KCS channel daemon（組裝完整 IP command）
   │  D-Bus: xyz.openbmc_project.IPMI.Message.execute(netfn, lun, cmd, data)
   ▼
phosphor-ipmi-host（command engine，執行命令）
   ├─ sensor / SDR：讀 D-Bus 上的 Sensor.Value 物件
   ├─ SEL：讀 phosphor-sel-logger 寫入的 SEL records
   ├─ FRU：讀 FRU EEPROM（i2c-fru）
   ├─ power：設 host power state（phosphor-state-manager）
   └─ user：phosphor-user-manager
   ▼  response 沿原 channel 寫回（KCS FIFO / RMCP+ IP response）
```

LAN 路徑：

```
ipmitool（client）── UDP 623 ──▶ phosphor-ipmi-net
   1. RMCP+ session：Open Session / RAKP 1–4（認證、cipher suite）
   2. 其後每個 IP packet：phosphor-ipmi-net 剝掉 session 層，
      把 NetFn/LUN/Cmd + data 用 D-Bus 轉給 phosphor-ipmi-host
      （xyz.openbmc_project.IPMI.Message.execute）
   3. response 包上 session 層回傳
   （SOL 也共用 port 623，作為另一種 session type）
```

重點：

- `phosphor-ipmi-net` 只負責 transport 與 session（RMCP+ 認證、cipher suite、SOL）；**命令執行在 `phosphor-ipmi-host`**——KCS 與 LAN 共用同一 command engine，行為一致。
- KCS channel 由 user-space daemon 處理（視平台，可能由 `phosphor-ipmi-host` 直接讀 KCS character device，或另有一個 `phosphor-ipmi-kcs` bridge daemon 負責 KCS over I2C 的情況）；讀完 KCS FIFO、組裝完整 IP command 後，同樣走 D-Bus Message API 執行。
- SDR list 從 D-Bus 上的 sensor 物件（`/xyz/openbmc_project/sensors/<Type>/<Name>`）動態產生；新增感測器自動多一條 SDR。
- SEL record 由 `phosphor-sel-logger` 產生：聽 D-Bus 上的 threshold alarm、state 改變訊號，轉成 SEL record 並持久化；IPMI SEL 命令（Get SEL Entry Information 等）從那讀。
- FRU EEPROM 由 `phosphor-fru-device` 讀取、parse 並 expose 到 D-Bus；IPMI Read FRU Data 讀的是 raw EEPROM。

## 相關專案 / daemon

- `phosphor-ipmi-host`：IPMI command engine 核心。KCS/BT channel、D-Bus Message API、SDR/SEL/FRU/sensor/power/user 等命令實作。
- `phosphor-ipmi-net`：IPMI over LAN（RMCP+、UDP 623）、session 管理、SOL。
- `phosphor-ipmi-kcs`：KCS channel bridge（部分平台 KCS 在 I2C mux 後面時使用）。
- `phosphor-sel-logger`：產生 SEL record（聽 D-Bus 事件、持久化）。
- `phosphor-fru-device`：讀取 / parse FRU EEPROM。
- `phosphor-state-manager`：host power state（IPMI power 命令的目標）。
- `phosphor-user-manager`：BMC user 與 role（IPMI user management 命令的目標）。

## D-Bus

中央 API（KCS 與 LAN 共用）：

- Service：`xyz.openbmc_project.IPMI`（`phosphor-ipmi-host` own）
- Object：`/xyz/openbmc_project/Ipmi`
- Interface：`xyz.openbmc_project.IPMI.Message`，method `execute(netfn, lun, cmd, data) → response data`

命令背後的「資料庫」大多是標準 OpenBMC D-Bus interface：

| 用途 | Interface / path 慣例 |
|---|---|
| Sensor 值 / threshold | `xyz.openbmc_project.Sensor.Value` + `Sensor.Threshold.*`，path `/xyz/openbmc_project/sensors/<Type>/<Name>` |
| Host power state | `xyz.openbmc_project.State.Host`（`CurrentHostState` / `RequestedHostTransition`） |
| FRU 欄位 | `phosphor-fru-device` expose 在 `/xyz/openbmc_project/...` 下 |
| SEL record | `phosphor-sel-logger` expose 在 `/xyz/openbmc_project/...` 下 |
| User | `phosphor-user-manager`（`/xyz/openbmc_project/User/...` 下的物件），帳密存於系統標準 account 檔 |
| Path 解析 | `xyz.openbmc_project.ObjectMapper`（`GetSubTree` 等） |

## 新手常問

- **IPMI 和 Redfish 差在哪？** IPMI 是 legacy 二進位協定；Redfish 是較新的 DMTF REST/JSON 介面。OpenBMC 兩者都實作，新開發優先 Redfish。
- **為什麼 LAN IPMI 要兩個 daemon？** `phosphor-ipmi-net` 只管 transport 與 session（RMCP+ 認證、SOL）；命令邏輯全在 `phosphor-ipmi-host`，讓 KCS 與 LAN 路徑行為一致。
- **KCS 還有存在的必要嗎？** 有。host BIOS POST、legacy firmware 工具、沒有網路的環境都走 KCS host channel。
- **SDR 怎麼來的？** 從 D-Bus sensor 物件動態產生；sensor 來去 SDR list 跟著變，不用手寫 SDR 檔。
- **SEL 為什麼是空的？** SEL record 由 `phosphor-sel-logger` 從 D-Bus 事件（threshold assert、state change 等）寫入；事件沒到 D-Bus 或 `phosphor-sel-logger` 沒跑就會缺記錄。
- **可以關掉 IPMI 嗎？** 可以。不部署 IPMI net/host daemon 即可（有些平台還要另行禁用 KCS 路徑）；Redfish 不受影響。
