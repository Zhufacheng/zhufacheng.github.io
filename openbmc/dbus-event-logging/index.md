# D-Bus event logging

OpenBMC 上很多 daemon 不是「自己硬寫 log/SEL」，而是**盯 D-Bus**：訂閱 D-Bus 訊號，收到後把它翻成一筆 elog entry 和/或一條 SEL record。這份文件講這個「monitor D-Bus → 產事件」的 pattern。

> 註：OpenBMC 裡這個 monitor 行為分散在**多個 daemon**（`phosphor-sel-logger`、`phosphor-logging`、`phosphor-state-manager`...），不是單一一個「dbus-monitor」daemon；下面講的是共同 pattern。

## 背景概念

- **D-Bus signal**：daemon 之間的廣播訊號。跟 method call 不同，signal 是 fire-and-forget、多對多。OpenBMC 常用的幾種：
  - `org.freedesktop.DBus.Properties.PropertiesChanged`：某個物件的 property 改變（sensor 值、threshold、狀態改變時）。
  - `org.freedesktop.DBus.ObjectManager.InterfacesAdded` / `InterfacesRemoved`：ObjectManager 子樹下新增/移除物件（新感測器/新 FRU 出現、消失）。
  - `org.freedesktop.DBus.Peer.NameOwnerChanged`：某個 well-known name 的 owner 改變（service 上線/掉線）。
  - 自訂 signal：daemon 自己發的（如 threshold 的 `HighAlarmAsserted`）。
- **Match rule**：D-Bus 的訂閱條件（`AddMatch`），可限定 signal 類型、member、sender、object path、interface，只收符合的訊號。
- **`sdbusplus`**：OpenBMC 的 C++ D-Bus 庫，提供訂閱 match + 回呼的 API，幾乎所有 monitor daemon 都用它。

## 怎麼運作

```
monitor daemon（如 phosphor-sel-logger）
   │
   ├─ 啟動時掛 match（AddMatch）：
   │     PropertiesChanged on /xyz/openbmc_project/sensors/...
   │     InterfacesAdded/Removed on /xyz/openbmc_project/...
   │     NameOwnerChanged on /org/freedesktop/DBus
   │
   ▼ （之後由訊號驅動，不 poll）
   收到 PropertiesChanged / InterfacesAdded / NameOwnerChanged / 自訂訊號
        │
        ▼
   handler：判斷這是什麼事件（哪顆 sensor、什麼 threshold、哪個 service）
        │
        ├─ 需要記 log → 呼叫 elog Create 寫一筆 Logging.Entry
        └─ 需要記 SEL → 查 SEL 設定、append 一條 SEL record（見 [SEL](../sel/)）
```

### 典型對應

| 收到的 D-Bus 訊號 | 翻成什麼 |
|---|---|
| sensor 的 threshold `HighAlarmAsserted` / 越限的 `PropertiesChanged` | elog entry + SEL Discrete Sensor Event（assert） |
| 感測器 `InterfacesAdded`（新 sensor/FRU 出現） | 「device present / 新感測器」log |
| 感測器 `InterfacesRemoved` | 「device lost」log / 值標 NaN |
| `NameOwnerChanged`（某 service 上線/掉線） | 狀態 log（如 host/BMC 狀態改變） |

重點是**事件驅動、不 poll**：monitor 只掛 match 等訊號，收到才反應；所以「sensor 變了 → log/SEL」是同步、即時的。

## 相關專案 / daemon

- `phosphor-sel-logger`：monitor D-Bus（elog entry + sensor 事件）→ SEL record（最典型的使用者）。
- `phosphor-logging`：elog，monitor 的「寫 log」目標。
- `phosphor-dbus-interfaces`：各 interface 定義（`.interface`），monitor 靠它知道要訂閱什麼。
- `phosphor-objmgr`（Object Mapper）：管理 D-Bus 物件樹、提供 `GetSubTree`/物件追蹤，常被 monitor 用來定位物件歸哪個 service。
- `sdbusplus`：訂閱 match / 收 signal 的 C++ 庫。
- 其他 monitor：`phosphor-state-manager`（host/BMC 狀態）等也各自訂閱 D-Bus。

## D-Bus

monitor 訂閱的主要 signal（標準 freedesktop 訊號 + OpenBMC 自訂）：

| signal | 從哪發出（interface / path） | 觸發時機 |
|---|---|---|
| `PropertiesChanged` | `org.freedesktop.DBus.Properties`，任意物件 path | property 改變（sensor 值、threshold、狀態） |
| `InterfacesAdded` | `org.freedesktop.DBus.ObjectManager` | 子樹下新物件出現 |
| `InterfacesRemoved` | `org.freedesktop.DBus.ObjectManager` | 物件被移除 |
| `NameOwnerChanged` | `org.freedesktop.DBus`，path `/org/freedesktop/DBus` | well-known name owner 改變（service 上/下線） |
| 自訂 alarm 訊號（如 `HighAlarmAsserted`） | 各 daemon 自訂 | 告警 assert/deassert |

match rule 範例（`AddMatch` 語法）：

```
type='signal',member='PropertiesChanged',interface='org.freedesktop.DBus.Properties',path='/xyz/openbmc_project/sensors/temperature/P1'
type='signal',member='InterfacesAdded',path_namespace='/xyz/openbmc_project/inventory'
type='signal',member='NameOwnerChanged',path='/org/freedesktop/DBus'
```

## 新手常問

- **這跟「poll」有什麼差別？** 不 poll。monitor 只掛 match、等 D-Bus 訊號，收到才跑 handler；即時且省 CPU。
- **一個 monitor 可以訂閱很多嗎？** 可以，掛多條 match rule 就行；handler 收到後各自處理。
- **為什麼要盯 `NameOwnerChanged`？** 知道某個 service 掉線（如 host 狀態、某 sensor daemon 掛掉），好記 log 或把相依的值標成不可信。
- **monitor 漏收訊號怎麼辦？** 啟動時先掛好 match 再開始處理（避免 service 起來前的事件漏掉）；有些也會先 `GetManagedObjects` 補一次現狀，再靠 signal 增量更新。
