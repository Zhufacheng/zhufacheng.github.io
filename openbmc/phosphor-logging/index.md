# phosphor-logging

`phosphor-logging`（簡稱 **elog**）是 OpenBMC 的事件記錄 service。其他 daemon 發生值得記的事（開機、sensor 越限、錯誤、firmware update...）時向它寫一筆 **log entry**；elog 把 entry 存成 D-Bus 物件並持久化到磁碟，必要時再當 IPMI SEL 的上游來源。

## 背景概念

- **log entry**：一筆事件。elog 把每筆 entry 當成一個 D-Bus 物件，有自己的 `Id`、`Severity`、`Message`、`Timestamp`、`AdditionalData`。
- **severity**：entry 的嚴重度，主要是 `Info` / `Warning` / `Error` / `Critical` 四級；決定事件重程度，也影響它要不要被升級成 SEL。
- **elog vs. SEL**：elog 是 OpenBMC 自己的結構化事件層（D-Bus 物件 + 磁碟持久化）；SEL 是 IPMI 標準的事件 log。elog 是「來源」，SEL 是其中一種「輸出」（由 `phosphor-sel-logger` 做映射）。

## 怎麼運作

### 1. 其他 daemon 寫 log

各 daemon 不自己開檔，而是呼叫 elog 的 **Create** API（D-Bus method）：傳入 severity + message（可附 additional data），elog 回傳新 entry 的 ID。寫 log 的 daemon 通常用 elog 附的 C/C++ helper library（libelog）或直接以 `sdbusplus` 呼叫 Create。

```
daemon A ──Create(sev, msg, data)──▶  /xyz/openbmc_project/Logging  (Service)
                                          │  分配 id
                                          ▼
                          /xyz/openbmc_project/Logging/entry/<id>
                          (interface: xyz.openbmc_project.Logging.Entry)
                                          │
                                          ▼
                          持久化到磁碟（/var/lib/phosphor/logging/，一 entry 一檔）
```

### 2. entry 物件

- 路徑 `/xyz/openbmc_project/Logging/entry/<id>`，interface `xyz.openbmc_project.Logging.Entry`。
- Properties：`Id`、`Severity`、`Message`、`Timestamp`、`AdditionalData`。
- Method `Delete`：移除該 entry。
- 讀取方（Redfish `LogService`、`ipmi` 工具、其他 daemon）直接 Get 這些 property，或訂閱 `InterfacesAdded`/`InterfacesRemoved` 看 entry 增刪。

### 3. 轉成 SEL（可選）

elog 本身只負責「記」。要不要升級成 IPMI SEL 由 `phosphor-sel-logger` 決定：它盯 Logging 的 entry 樹（及 sensor 事件訊號），依設定檔把符合規則的 event 翻成一條 SEL record 寫進 SEL 檔。詳見 [SEL](../sel/)。

## 相關專案 / daemon

- `phosphor-logging`：elog service 本體。
- `phosphor-sel-logger`：把 elog entry / D-Bus 事件映射成 IPMI SEL record。
- `phosphor-dbus-interfaces`：D-Bus interface 定義（含 `Logging.*` 的 `.interface` 檔）。
- `bmcweb`：把 elog / SEL 暴露成 Redfish `LogService`。

## D-Bus

| 項目 | 值 |
|---|---|
| well-known service | `xyz.openbmc_project.Logging` |
| service 物件路徑 | `/xyz/openbmc_project/Logging` |
| entry 物件路徑 | `/xyz/openbmc_project/Logging/entry/<id>` |
| service interface | `xyz.openbmc_project.Logging.Service`（`Create` method） |
| entry interface | `xyz.openbmc_project.Logging.Entry` |
| entry properties | `Id`、`Severity`、`Message`、`Timestamp`、`AdditionalData` |
| entry method | `Delete` |

`Create` 回傳新 entry 的 `Id`；entry 建立後會發 `InterfacesAdded`，讓訂閱方（`phosphor-sel-logger`、Redfish...）知道有新 entry。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/var/lib/phosphor/logging/` | entry 持久化目錄（一筆 entry 一個檔，檔名為 entry ID） |
| `xyz.openbmc_project.Logging.Entry.interface` | entry 的 D-Bus interface 定義，出自 `phosphor-dbus-interfaces`（build 時裝進系統） |

## 新手常問

- **其他 daemon 怎麼記 log？** 呼叫 `Create(severity, message, additionalData)`（通常透過 libelog helper），拿到 entry ID，不用自己開檔。
- **severity 有哪些？** 主要 `Info` / `Warning` / `Error` / `Critical`；越嚴重越可能升級成 SEL / 觸發告警。
- **elog 跟 SEL 的關係？** elog 是來源（D-Bus 物件 + 磁碟），SEL 是其中一種輸出；是否升級成 SEL 由 `phosphor-sel-logger` 的設定決定，不是每筆 elog 都會進 SEL。
- **怎麼讀 log？** 直接 Get entry 的 property、訂閱 `InterfacesAdded`，或走 Redfish `LogService` / `ipmi` 工具。
- **entry 會一直累積嗎？** 會持久化到磁碟；通常有 max entries 上限，滿時清最舊的（行為依平台/設定）。
