# Power capping

Power capping 是把系統實際功耗限制在一個設定上限（cap）以內的機制，用於資料中心的 power budget 管理（機房供電限制、PUE、合規）。在 OpenBMC 裡，**Redfish 的介面定義是標準化的**（Chassis Power 資源裡的 `PowerLimit`），但「BMC 怎麼讀實際功耗、怎麼叫平台真的降功耗」在**上游沒有統一標準化實作**——各平台自己實作（獨立 daemon、CPLD / PSU 內建、或 host firmware 協作），機制是 platform-dependent。

## 背景概念

- **Power budget / capping**：資料中心對單機給出的電力量（budget）；cap 是使用者設定的功耗上限。實際功耗超過 cap 時，平台要主動降負載，所以這是一個**閉環控制**，不是一次性設定。
- **CAP（capping authority）**：Redfish 規格區分 cap 的來源——`Admin`（系統管理者設）與 `CAP`（資料中心上層系統下達）；BMC 可能收到外部 cap authority 下來的限制，並回報生效值。
- **PowerLimitType / PowerLimitState**：`PowerLimitType` 標記這組限制的類型（`Admin` / `CAP` / `CAPAndAdmin`）；`PowerLimitState` 是 cap 的開關（`On` / `Off`）。
- **Enforcement（執行）**：真正「降功耗」的動作發生在 host / 平台側——通常是 host firmware 調低 CPU 的 power limit，或平台的電源管理硬體直接介入。BMC 本身不消耗 host 的功耗，它只負責量測、決策與下達訊號。

## 怎麼運作

通用閉環（具體實作因平台而異，以下是常見形狀）：

```
使用者 / 資料中心下達 cap
        │  Redfish PATCH /redfish/v1/Chassis/{id}/Power
        │  PowerControl[].PowerLimit：MaxPowerWatts、PowerLimitType、
        │  PowerLimitState（On/Off）、CorrectionFactor
        ▼
bmcweb ── 寫到 D-Bus（平台 capping 元件自己定義的物件 / interface）
        ▼
平台 power capping 邏輯（獨立 daemon / CPLD / PSU 內建，平台相關）
        │  ① 讀當前實際功耗（PMBus power 感測器 / 獨立 power meter）
        │  ② 實際功耗 > cap → 對平台下達「降功耗」訊號
        │     （經 host firmware 通道，或 CPLD 直接介入）
        ▼
host / 平台降功耗（CPU power limit ↓、負載 ↓）
        │
        └── 實際功耗回落至 cap 以下；週期取樣，形成閉環
```

1. **設定**：cap 值與開關經 Redfish 進來；bmcweb 把它寫到平台 capping 元件暴露的 D-Bus 物件上（介面名稱平台自訂）。
2. **量測**：BMC 讀實際功耗，來源通常是 `phosphor-psu-manager` 發佈的 PSU power 感測器（PMBus 的 input / output power），或獨立的 power meter（`/xyz/openbmc_project/sensors/power/` 下的 `power` 型感測器）。
3. **執行**：超過 cap 時由平台機制介入，把功耗壓回上限內。具體通道（host firmware 指令、CPLD register、PSU 指令）是 platform-dependent。
4. **回饋**：生效的 cap、當前實際功耗（`PowerConsumedWatts`）、cap 是否 active 回報到 Redfish，供上層與使用者確認閉環結果。

## 相關專案 / daemon

- `bmcweb`：Redfish Chassis Power / `PowerLimit` 的 surface（介面定義來自 Redfish 規格）
- `phosphor-psu-manager`：實際功耗感測器的常見來源（見同 wiki 該頁）
- 平台 power capping 元件：**上游沒有統一的 repo / daemon**；實作可能是廠商 daemon、CPLD 功能或 host firmware 的一部分。要接這功能，先確認目標平台上 capping 由誰負責。

## D-Bus（通用 surface）

因為沒有標準化的 interface 名稱，以下是**常見形狀**，具體名稱以平台實作為準：

| 項目 | 說明 |
|---|---|
| cap 設定物件 | 平台 capping 元件暴露一個 D-Bus 物件，property 為 cap 值 / 開關 / 類型 |
| 功耗讀取 | `xyz.openbmc_project.Sensor.Value`（`/xyz/openbmc_project/sensors/power/<name>`） |
| 呼叫端 | bmcweb 經 ObjectMapper 定位 capping 物件，不 hard-code service 名 |

> 不知道某個平台用什麼介面時，最可靠的辦法：看該平台 capping 元件在 D-Bus 上暴露哪些物件，或追 bmcweb 的 chassis power 程式碼路徑。

## 新手常問

- **OpenBMC 有標準的「power capping daemon」嗎？** 沒有。上游沒有統一實作；平台沒提供對應元件時，Redfish 的 `PowerLimit` 相關欄位會沒有值 / 不可用。
- **Redfish 改了 cap，為什麼功耗沒降？** 查閉環三環：設定有沒有真的寫下去（D-Bus property 有沒有變）、實際功耗有沒有真的超過 cap、平台的執行通道（host firmware / CPLD）有沒有動作。
- **實際功耗怎麼量？** 通常就是 PSU 的 PMBus power 感測器（`phosphor-psu-manager` 發佈）；精度與更新率受感測器來源限制。
- **Capping 是一次性設定嗎？** 不是。cap 值是穩態設定；「降功耗」是持續的封閉迴圈動作，只在超過 cap 時被觸發。
- **`PowerLimitType` 為什麼要分 Admin / CAP？** 區分限制來源：管理員自己設的上限 vs. 資料中心 CAP 下來的上層限制；兩者同時存在時用 `CAPAndAdmin`，生效值取較嚴格的。
