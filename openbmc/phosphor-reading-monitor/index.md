# phosphor-reading-monitor

`phosphor-reading-monitor` 盯住 D-Bus 上的感測器值，跟 threshold（high / low，含 hysteresis）比較，越線就發 threshold alarm event（elog + IPMI SEL）。它是**事件驅動**的——訂閱 `Sensor.Value` 的 `PropertiesChanged`，值一變就判定，不是排程輪詢。它是整組感測器裡負責「告警」的那一環，本身不讀硬體、不產生感測器。

## 背景概念

- **感測器 D-Bus 物件**：值在 `/xyz/openbmc_project/sensors/<Type>/<Name>`，interface `xyz.openbmc_project.Sensor.Value`（`Value` 是關鍵）。
- **Threshold interface**：`xyz.openbmc_project.Sensor.Threshold.<Severity>`，`Severity` ∈ `Warning` / `Critical` / `SoftShutdown` / `HardShutdown` / `PerformanceLoss`。每個有 `High` / `Low` 兩條線，外加 `HighHysteresis` / `LowHysteresis` 兩個回差值。
- **Hysteresis（回差）**：避免值在臨界點附近抖動時 alarm 一直亮滅。越線即 assert，但要**退回線內回差距離以內**才 deassert。
- **elog / SEL**：告警的兩類落地。elog = `phosphor-logging` 的記錄；SEL = IPMI System Event Log（`phosphor-ipmi-sel` 管的那份）。

## 怎麼運作

```
systemd 起 phosphor-reading-monitor.service
  │
  ├─ 建 D-Bus 連線
  ├─ 掛 match：盯 /xyz/openbmc_project/sensors 下
  │           所有 Sensor.Value 的 PropertiesChanged
  │     （也留意感測器 / threshold interface 的出現與消失）
  ├─ request_name(...) 並進 event loop
  │
  ▼ 之後（全由 D-Bus 訊號驅動，沒有 poll）
  某顆感測器 Value 改變
         │
         ▼
  PropertiesChanged handler
     ├─ 取該感測器各 Severity 的 High / Low / hysteresis
     ├─ 跟新值比較：
     │     越 High（或 Low）        → assert（若還沒亮）
     │     亮著且退回 hysteresis 內 → deassert
     └─ assert / deassert 時：
           ├─ 發 elog（phosphor-logging）
           └─ 發 IPMI SEL record
```

1. **訂閱**：daemon 起來先掛 match 規則，訂閱 `Sensor.Value` 的 `PropertiesChanged`（並追蹤感測器與 threshold interface 的增刪）。因為是訂閱，新值一上來就觸發，**沒有排程 poll**。
2. **取 threshold**：對每顆要判定的感測器，讀各 Severity 的 `High` / `Low` 與對應 hysteresis。
3. **判定（assert）**：`value >= High`（或 `value <= Low`）且該方向 alarm 還沒亮 → **assert**，記下狀態。
4. **判定（deassert）**：alarm 亮著、`value < High - HighHysteresis`（或 `value > Low + LowHysteresis`）→ **deassert**。
5. **落地**：assert / deassert 各產生一條 event——elog 記錄（走 `phosphor-logging`）與 IPMI SEL record（走 SEL 子系統）。

assert / deassert 的對稱式：

- High 線：assert 條件 `value >= High`；deassert 條件 `value < High - HighHysteresis`。
- Low 線：assert 條件 `value <= Low`；deassert 條件 `value > Low + LowHysteresis`。

## 相關專案 / daemon

- `phosphor-reading-monitor` — 本 daemon
- `phosphor-logging` — elog 落地（`xyz.openbmc_project.Logging.Error` 一系的 log 介面）
- `phosphor-ipmi-sel` — IPMI SEL 記錄的維護
- 各感測器 daemon（`phosphor-hwmon-sensor`、`phosphor-virtual-sensor` 等）— 提供被監視的 `Sensor.Value`

## D-Bus

- **訂閱**：match 盯 `/xyz/openbmc_project/sensors` 下所有 `xyz.openbmc_project.Sensor.Value` 的 `PropertiesChanged`；另追蹤感測器 / threshold 物件的 `InterfacesAdded` / `InterfacesRemoved`。
- **讀取的 interface**（判定用）：
  - `xyz.openbmc_project.Sensor.Value` → `Value`
  - `xyz.openbmc_project.Sensor.Threshold.<Severity>` → `High`、`Low`、`HighHysteresis`、`LowHysteresis`
- **寫出**：elog 走 `phosphor-logging` 的 log 介面；SEL 走 SEL 子系統介面。（具體 method 名隨版本而異，概念上是「新增一筆 log / SEL」。）
- daemon 本身也註冊 `xyz.openbmc_project.*` well-known name 來持有連線；但它的主要角色是**消費者**，不是感測器提供者。

## 新手常問

- **它怎麼知道 threshold 值？** 從感測器自己掛的 `Sensor.Threshold.<Severity>` interface 讀，不是自己另外維護一份設定。感測器沒掛 threshold，就不會判定。
- **為什麼要 hysteresis？** 不給回差的話，值在臨界點抖動會讓 alarm 反覆亮滅、刷爆 log 與 SEL。回差讓 assert 和 deassert 用不同條件，形成一個「死區」。
- **它會改感測器值嗎？** 不會。只讀 `Value` 與 threshold，只寫 elog / SEL。
- **五種 Severity 差別在哪？** 嚴重度遞增（`Warning` → `Critical` → `PerformanceLoss` / `SoftShutdown` → `HardShutdown`），對應不同的應對動作與 log / SEL 事件級別；判定邏輯一樣，只是各自的 high / low 線不同。
- **感測器一閃就消失會怎樣？** 物件移除（`InterfacesRemoved`）時對應的監視狀態會被清掉，不會留下鬼狀態。
