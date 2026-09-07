# phosphor-hwmon-sensor

`phosphor-hwmon-sensor` 把 Linux 的 hwmon 驅動（sysfs `/sys/class/hwmon/`）暴露成標準 OpenBMC D-Bus 感測器。它做的事很單純：掃 hwmon 裝置、讀 channel、依 hwmon 屬性前綴判定感測器類型（temperature / voltage / current / power / fan_tach），再以標準 `Sensor.Value` 物件掛上 D-Bus。對其他 daemon（`phosphor-reading-monitor`、`bmcweb`、`phosphor-ipmi-sensor`）而言，它就是一顆普通感測器，不知道背後是 hwmon。

## 背景概念

- **hwmon**：Linux 的硬體監控框架。驅動把感測器通道暴露到 `/sys/class/hwmon/hwmon*/`，每個裝置一個子目錄，裡面是一堆 `*_input` 檔案。
- **hwmon channel 命名慣例**（決定類型判定）：
  - `temp1_input` → 溫度（milli-°C）
  - `in1_input` → 電壓（mV）
  - `curr1_input` → 電流（µA）
  - `power1_input` → 功率（µW）
  - `fan1_input` → 風扇轉速（RPM）
  - 對應的 `temp1_label`、`in1_label` 等 label 檔（若有）決定感測器顯示名。
- **感測器 D-Bus 物件**：一律 `/xyz/openbmc_project/sensors/<Type>/<Name>`，interface `xyz.openbmc_project.Sensor.Value`（`Value`、`Unit`、`MinValue`/`MaxValue`）。

## 怎麼運作

```
systemd 起 phosphor-hwmon-sensor.service
  │
  ├─ 建 D-Bus 連線、Object Manager 管 /xyz/openbmc_project/sensors 子樹
  ├─ 掃描 /sys/class/hwmon/hwmon*
  │     對每個裝置：
  │       ├─ 讀 name（驅動名，如 nct75 / max16071 / pmbus）
  │       ├─ 列目錄，找所有 <prefix><N>_input
  │       └─ 依前綴建對應類型感測器
  ├─ request_name(...) 並進 event loop
  │
  ▼ 之後
  讀 input 檔 → 換算單位 → 寫 Sensor.Value
  新 hwmon 裝置出現 / 消失 → 建立 / 移除對應感測器物件
```

1. **掃描**：`main()` 起來後列 `/sys/class/hwmon/hwmon*`。對每個子目錄，讀 `name` 取得驅動名，再列目錄內容找出所有 `<prefix><N>_input` 檔。
2. **類型映射**：依屬性前綴決定感測器類型與物理單位——`temp`→`temperature`、`in`→`voltage`、`curr`→`current`、`power`→`power`、`fan`→`fan_tach`。
3. **命名**：優先用 hwmon 的 `*_label`；沒有 label 就退回「驅動名 + 通道編號」。
4. **換算**：hwmon 的 input 值固定用 milli/micro 單位（milli-°C、mV、µA、µW），daemon 換算成感測器大單位（°C、V、A、W）後寫入 `Value`。
5. **發布**：每顆 channel = 一個 D-Bus 物件，經 Object Manager 掛上 `/xyz/openbmc_project/sensors/<type>/<name>`，interface `Sensor.Value`。
6. **動態**：新 hwmon 裝置被驅動註冊（或卸除）時，daemon 偵測並建立 / 移除對應感測器物件。

## 相關專案 / daemon

- `phosphor-hwmon-sensor` — 本 daemon
- `phosphor-sensors` — 舊一代 hwmon / eeprom 感測器 daemon（同 lineage 的前身）
- 上游 hwmon 驅動（`nct75`、`max16071`、`pmbus` 等）— 提供 sysfs channel
- `phosphor-reading-monitor` — 消費這些感測器做 threshold 告警
- `bmcweb` / `phosphor-ipmi-sensor` — 對上層（REST / IPMI）呈現這些感測器

## D-Bus

- **Object path**：`/xyz/openbmc_project/sensors/<Type>/<Name>`，`Type` ∈ `temperature` / `voltage` / `current` / `power` / `fan_tach`；由 Object Manager 管理、可被 `GetManagedObjects` 一次列舉。
- **Interface**：`xyz.openbmc_project.Sensor.Value`
  - `Value` (double) — 換算後數值
  - `Unit` (string) — 物理單位
  - `MinValue` / `MaxValue` (double) — 有效範圍（驅動 / 設定有提供才設）
- **Well-known service**：daemon 註冊一個 `xyz.openbmc_project.*` 的 well-known name 來擁有該子樹（hwmon 感測器 lineage 慣用 `xyz.openbmc_project.HwmonSensors`）。客戶端通常以 object path 發現感測器，而非綁定 service name。
- 若該 channel 有 threshold 設定，可另掛 `xyz.openbmc_project.Sensor.Threshold.<Severity>`（判定與告警由 `phosphor-reading-monitor` 負責，見該頁）。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/sys/class/hwmon/hwmon*/` | hwmon 裝置根目錄（每裝置一子目錄） |
| `/sys/class/hwmon/hwmonN/name` | 驅動 / 裝置名 |
| `/sys/class/hwmon/hwmonN/<prefix><C>_input` | 通道原始值（temp / in / curr / power / fan） |
| `/sys/class/hwmon/hwmonN/<prefix><C>_label` | 通道顯示名（可選） |
| `/xyz/openbmc_project/sensors/...`（D-Bus） | 本 daemon 發布的感測器物件 |

## 新手常問

- **跟 `phosphor-sensors` 是什麼關係？** 同 lineage：把 hwmon（與 eeprom）通道變 D-Bus 感測器。`phosphor-hwmon-sensor` 是專注 hwmon、較通用的版本；老部署常用 `phosphor-sensors`。兩者輸出都是標準 `Sensor.Value` 物件。
- **為什麼有的 hwmon 裝置沒被暴露？** 只有 temp / in / curr / power / fan 前綴會被映射；其他屬性（`fan*_mode`、`pwm*`、`alarm*`、`crit*` 等）不是感測器通道，會被忽略。
- **單位是哪裡來的？** hwmon sysfs 固定用 milli/micro 單位，daemon 換算成 `Sensor.Value` 的大單位，`Unit` 欄位記載最終單位。
- **感測器名為什麼有時候是 label、有時候是驅動名 + 編號？** 有 `*_label` 就用 label，否則退回驅動名 + 通道號。
- **它會自己算 threshold 嗎？** 不會。它只負責「量」與發布 `Value`；threshold 判定與告警由 `phosphor-reading-monitor` 負責。
