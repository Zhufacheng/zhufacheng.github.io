# SEL (System Event Log)

**SEL（System Event Log）** 是 IPMI 標準的 out-of-band 事件 log：一連串帶 timestamp 的 event record，記 sensor 越限、開關機、錯誤等事件。它是 BMC 最經典的事件記錄，IPMI 命令、本機 `ipmi-sel` 工具、Redfish `LogService` 都能讀。

## 背景概念

- **SEL record**：SEL 的基本單位。每條 record 有：唯一遞增的 **Record ID**（2 bytes，接近 0xFFF0 時 wrap）、**Record Type**、**Timestamp**（3-byte IPMI SEL 時間）、**Generator ID**、一段 **Event Message**。
- **最常見的 record type** 是 **Discrete Sensor Event**（type `0x02`）：Event Message 內含 Sensor Type、Sensor Number、**Event Type**（含 1 bit 表示 assert/deassert 方向）、3 bytes Event Data。開關類事件（斷言/解除）靠這個方向 bit 分辨。
- **SEL Time**：SEL 用**自己的 3-byte 時間**（以 `Set SEL Time` 設定的 UTC offset），**不直接讀系統時鐘**。所以看 SEL 時間要補這個 offset。
- **Generator ID**：標記「誰」產生這筆事件（通常指向 BMC 或某顆 sensor）。

## 怎麼運作

### 1. 產生 SEL record

OpenBMC 主要靠 `phosphor-sel-logger` 產生 SEL，兩類事件來源：

- **elog entry**：盯 `phosphor-logging` 的 entry（`Logging.Entry` 物件建立時）。
- **sensor / D-Bus 事件**：訂閱感測器的 threshold assert/deassert 訊號與相關 signal。

事件進來後，`phosphor-sel-logger` 查它的 **JSON 設定檔**（把「某種事件」映射到「該寫哪條 SEL record」：sensor、event type、event data、description），符合就 **append 一條 IPMI SEL record 進 SEL 檔**。

```
daemon 記 elog / sensor threshold assert
      │  (D-Bus: Logging.Entry 建立 / 告警訊號)
      ▼
phosphor-sel-logger
      │  比對 JSON 設定（事件 → SEL record 模板）
      ▼
符合規則 → 產生 Discrete Sensor Event record
      ▼
append 進 SEL 檔（/var/lib/ipmi/sel，二進位）
```

### 2. 讀 SEL

- **IPMI 命令**（`ipmitool` / 本機 `ipmi`）：
  - `Get SEL Entry`（cmd `0x40`）：依 Record ID 取一筆。
  - `Get SEL Info`（`0x43`）：取 SEL 大小、entry 數、最早/最晚 record。
  - `Get SEL Time`（`0x48`）/ `Set SEL Time`（`0x49`）：讀/設 SEL 時間 offset。
  - `Clear SEL`（`0x47`）：清 SEL。
- **本機工具**：`ipmi-sel list` / `ipmi-sel print`（直接對 SEL 檔操作）。
- **Redfish**：`bmcweb` 把 SEL 暴露成 `LogService`，可列出/取/清 record；跟 IPMI 看的是同一份 SEL。

## 相關專案 / daemon

- `phosphor-sel-logger`：把 elog entry / D-Bus 事件映射成 SEL record、寫 SEL 檔。
- `phosphor-logging`：elog（SEL 的上游事件來源之一）。
- `ipmi-sel` 等 IPMI 工具：SEL 檔讀寫 / IPMI SEL 命令。
- `bmcweb`：Redfish `LogService`（SEL 的 Redfish 視圖）。

## D-Bus

SEL 本身是 IPMI/磁碟概念、**不是** D-Bus 物件；主要靠 IPMI 命令與 Redfish 讀。D-Bus 只出現在「產生端」：`phosphor-sel-logger` 訂閱 `phosphor-logging` 的 `Logging.Entry`（`/xyz/openbmc_project/Logging/entry/<id>`）與感測器的 threshold/告警訊號，把這些事件轉成 SEL record。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/var/lib/ipmi/sel` | SEL 本體（二進位，IPMI SEL 格式） |
| `phosphor-sel-logger` 的 SEL 設定 JSON | 「事件 → SEL record」映射（sensor、event type、event data、description），裝到系統設定目錄 |

## 新手常問

- **SEL 時間看起來怪怪的？** SEL 用 IPMI SEL 自己的 3-byte 時間 + `Set SEL Time` 的 offset，不是直接讀系統 RTC；算真實時間要補 offset。
- **為什麼某筆 elog 沒進 SEL？** 只有被 `phosphor-sel-logger` 設定命中的 event 才寫成 SEL；elog 是全集，SEL 是被映射的一小部分。
- **assert 跟 deassert 怎麼分？** Discrete Sensor Event 的 Event Type 高 bit 是方向：`1` = assert（事件發生）、`0` = deassert（解除）。
- **Redfish 跟 IPMI 看的 SEL 一樣嗎？** 一樣，同一份 `/var/lib/ipmi/sel`，`bmcweb` 只是提供 Redfish 視圖。
- **清 SEL 怎麼做？** `ipmitool sel clear`（IPMI `Clear SEL`）、本機 `ipmi-sel` 工具，或 Redfish 的 Clear LogService 動作。
