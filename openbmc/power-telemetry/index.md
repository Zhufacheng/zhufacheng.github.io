# Power telemetry

OpenBMC 的電力量測（power telemetry）沒有「單一主導 daemon」。功率值可能從 hwmon power channel、I2C 功率感測器、或 host 的 PLDM sensor 任一路進來，最後都匯成標準 `power` 感測器（`Sensor.Value`，單位瓦特）掛上 D-Bus；上層的功率限制（power capping）與報表都讀這顆 D-Bus 感測器。這一頁講「功率值從哪來、到 D-Bus 長什麼樣、誰在用」。

## 背景概念

- **功率感測器 D-Bus 物件**：`/xyz/openbmc_project/sensors/power/<Name>`，interface `xyz.openbmc_project.Sensor.Value`（`Value` = 瓦特、`Unit`、`MinValue`/`MaxValue`）。這是各路功率來源的**統一出口**——上層不關心跳的是哪顆硬體。
- **Energy（能量）累積**：功率對時間積分（W × 秒 → Wh）得到能量。有些平台會累積「自開機以來用了多少 Wh」這類計數，供報表 / 能耗統計。
- **Power capping（功率限制）**：讀當前功率，若超過上限就降頻 / 限電；讀值來自同一顆 `power` 感測器。

## 功率值從哪來（三大來源）

```
來源 A：hwmon power channel            來源 B：I2C / PMBus 功率感測器   來源 C：host PLDM sensor
  /sys/class/hwmon/hwmon*/power1_input    專用的功率監控 IC                 host 端透過 PLDM 上報
        │                                       │                                │
        ▼                                       ▼                                ▼
  phosphor-hwmon-sensor              （常也走 hwmon 的 pmbus driver，          plmd / PLDM sensor 端
  （讀 power1_input，µW→W）            所以最終也落到來源 A 這條路）            （host → BMC）
        │                                       │                                │
        └──────────────────────┬────────────────┴───────────────┬────────────────┘
                               ▼                                ▼
                 /xyz/openbmc_project/sensors/power/<Name>   （同一標準 Sensor.Value）
                               │
                               ▼
            phosphor-reading-monitor（告警）/ power capping（限制）/ 報表（能耗統計）
```

- **來源 A：hwmon power channel** — 最常見。功率監控 IC（PMBus 的 `POWER_INPUT` 等）經 `pmbus` 驅動暴露成 `power1_input`（µW），由 `phosphor-hwmon-sensor` 讀出來換算成瓦特。
- **來源 B：I2C / PMBus 功率感測器** — 專用功率監控 IC。很多時候它同樣是經 `pmbus` 驅動落成 hwmon `power*` channel，最終跟來源 A 合併；少數平台用獨立 daemon 直接讀 I2C。
- **來源 C：host PLDM sensor** — host 端的功率值透過 PLDM（host 的 PLDM sensor 端）上報給 BMC，再暴露成 D-Bus 感測器。用於「host 自己量、BMC 只收」的場景。

> 實際部署通常是「`phosphor-hwmon-sensor` 提供功率 + （需要的話）PLDM」的組合，而不是某一個專門的 power daemon。

## 怎麼運作（概念 + D-Bus）

1. **採樣**：依來源不同，功率值來自 hwmon sysfs、I2C 讀取、或 PLDM 訊息。
2. **換算**：統一換算成瓦特（hwmon 的 µW、PMBus 的原始碼值都要換）。
3. **發布**：掛成標準 `power` 感測器，`/xyz/openbmc_project/sensors/power/<name>`，interface `Sensor.Value`。
4. **累積 / 消费**：
   - **能量累積**：對功率做時間積分（Wh），供能耗統計（有累積邏輯的平台才做）。
   - **Power capping**：讀 `Value` 跟 cap 上限比，超過就觸發限制動作。
   - **告警**：`phosphor-reading-monitor` 依 threshold（如過功率）發 alarm。
   - **報表 / REST**：`bmcweb` 等對上層呈現。

## 相關專案 / daemon

- `phosphor-hwmon-sensor` — 讀 hwmon `power*` channel（來源 A / B 的主要出口）
- 上游 `pmbus` 驅動 — 把功率監控 IC 暴露成 hwmon
- `pldmd` / PLDM sensor 端 — host 端功率上報（來源 C）
- `phosphor-reading-monitor` — 功率 threshold 告警
- power capping 相關 daemon — 讀功率做限制
- `bmcweb` — 對上層（REST / 報表）呈現

## D-Bus

- **Object path**：`/xyz/openbmc_project/sensors/power/<Name>`。
- **Interface**：`xyz.openbmc_project.Sensor.Value`
  - `Value` (double) — 當前功率（瓦特）
  - `Unit` (string) — 功率單位
  - `MinValue` / `MaxValue` (double) — 有效範圍（有提供才設）
- **Threshold**（可選）：`xyz.openbmc_project.Sensor.Threshold.<Severity>`，供 `phosphor-reading-monitor` 做過功率告警。
- **能量**：若有能量累積，通常是另一顆 `energy` 感測器（`/xyz/openbmc_project/sensors/energy/<Name>`）或額外 property 呈現，單位 Wh——具體取決於平台實現。
- 因來源不同，提供這棵子樹的 well-known service 也會不同（hwmon 路徑下常為 `xyz.openbmc_project.HwmonSensors` 一系）；**以 object path 發現功率感測器最穩**。

## 新手常問

- **為什麼找不到「功率 daemon」？** 因為功率沒有專屬主導 daemon——hwmon 路徑由 `phosphor-hwmon-sensor` 出、host 路徑由 PLDM 出。找功率就找 `/xyz/openbmc_project/sensors/power/` 子樹。
- **功率跟能量是什麼關係？** 功率是即時值（W），能量是功率對時間的積分（Wh）。同一顆功率感測器可以喂能量累積，但能量本身常另立 `energy` 感測器。
- **Power capping 讀哪裡？** 讀同一顆 `power` 感測器的 `Value`，跟 cap 上限比；跟量測是同一份 D-Bus 資料。
- **怎麼判斷我的功率走哪條路？** 看 `/sys/class/hwmon/` 有沒有 `power*` channel（走 hwmon）；或看 host 是否啟用 PLDM sensor。兩者可並存，名字不同而已。
- **單位一定要是瓦特嗎？** 標準 `power` 感測器以瓦特呈現；hwmon / PMBus 原始值（µW 等）在 daemon 層已換算，客戶端拿到的就是 W。
