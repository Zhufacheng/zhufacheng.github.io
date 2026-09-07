# phosphor-power

`phosphor-power` 是 OpenBMC 的 host 電源管理 daemon：負責 host 開機 / 關機 / 循環上電 / 重設，依照硬體設計把 power sequence 一步一步執行（上電順序、等待、依賴），並監控完成與故障。host 目前電源狀態統一以 `/xyz/openbmc_project/state/host0` 這個標準 D-Bus 物件發佈，Redfish / IPMI 等前端把使用者的電源動作轉成對它的 D-Bus method call。

本文對應上游 `openbmc/phosphor-power` repo（`phosphor-power` daemon 與 `power-restore-policy` daemon 都在這個 repo 裡）。

## 背景概念

- **狀態（state）與遷移（transition）**：`CurrentHostState` 是字串列舉：`Off`、`On`、`Quiesced`、`TransitionToOff`、`TransitionToOn`、`Restarting`。Transition 狀態只在 sequence 執行期間存在，之後會落到穩態（`On` / `Off`）。
- **Power sequence（sequencing）**：真實硬體不是「拉一個 pin」就開機，要依序做數個步驟（備用電源使能、release reset、拉 power-good…）。每個步驟是 GPIO 或 CPLD register 操作，帶 delay，步驟之間可以互相依賴。sequence 用 JSON config 描述，daemon 逐步執行。
- **PWRGD（power good）**：硬體給的「host 已上電」訊號（GPIO 或 CPLD status bit）。daemon 發出上電指令後要等 PWRGD 才能確認遷移完成；等不到（timeout）就判定故障。
- **Power restore policy**：BMC 重開機 / 供電恢復後，host 該回到什麼狀態（常開 / 常關 / 恢復先前的狀態）。

## 怎麼運作

### 請求路徑（前端 → D-Bus → daemon → 硬體 → 狀態回傳）

```
Redfish (bmcweb) / IPMI (phosphor-ipmi-host)
  e.g., Reset=On / ipmitool chassis power on
        │  轉成 D-Bus method call
        ▼
service  xyz.openbmc_project.State.Host
object   /xyz/openbmc_project/state/host0
method   PowerOn / PowerOff / PowerCycle / WarmReset
        │
        ▼
phosphor-power（event loop）
  ├─ 狀態檢查（非法遷移拒絕或去重）
  ├─ CurrentHostState → TransitionToOn（或 TransitionToOff）
  ├─ 依 JSON config 執行 sequence：
  │     step: 寫 GPIO（libgpiod）/ CPLD register、delay、等依賴
  ├─ 等 PWRGD 確認（timeout → log 故障、停在目前狀態）
  └─ CurrentHostState → On / Off
        │  PropertiesChanged 訊號
        ▼
D-Bus 上所有 reader（bmcweb、IPMI、其他 daemon）讀到同一個新狀態
```

1. **請求進來**：bmcweb 把 Redfish 的 `ComputerSystem.Reset`（On / Off / ForceOff / GracefulShutdown / GracefulRestart / WarmReset…）映射到對應的 D-Bus method；IPMI chassis control command（0x28）由 `phosphor-ipmi-host` 轉成同一組 call。Graceful 類動作走 host 端協調（ACPI / IPMI 電源指令），不直接拉硬關機。
2. **狀態機推進**：daemon 先更新 `CurrentHostState` 到 transition 狀態，外部 reader 從 D-Bus 就能看到「正在進行中」。
3. **硬體動作**：sequence 的每個 step 是 GPIO 寫入（經 libgpiod）或 CPLD register 寫入（平台相關）；step 之間可以有 delay 與依賴關係。
4. **完成確認**：上電後等 PWRGD；`ChassisPowerTimeouts`（`State.Host` 的 property）定義遷移 timeout。超時判定故障——log error、把狀態停在 transition 狀態，不會硬推成 `On`。
5. **結果寫回**：sequence 結束寫回穩態（`On` / `Off`）並發 `PropertiesChanged`；Redfish 的 state、`ipmitool chassis status` 之類全部讀同一個 property，不會各自維護一份。

### Power restore policy

`power-restore-policy`（同 repo 的獨立小 daemon）在 BMC 啟動時依 policy 決定要不要自動把 host 開起來：

- `AlwaysRestore`：跟 BMC 關機前 host 的狀態
- `AlwaysOff`：不自動開機
- `PreviousState`：恢复到掉電前的狀態（需要把該位元持久化，平台相關）

## 相關專案 / daemon

- `phosphor-power`（本 repo）：daemon 本體 + C++ 函式庫（其他 daemon 可以 link 它直接下電源指令，不必自己寫 D-Bus call）
- `power-restore-policy`（同 repo）：開機時的 power restore 決策
- `bmcweb`：Redfish 前端，映射 Reset / 電源 action
- `phosphor-ipmi-host`：IPMI 前端（chassis control）
- `entity-manager`：部分平台的電源硬體描述 / GPIO 定義來源
- `phosphor-logging`：故障事件進 event log

## D-Bus

| 項目 | 值 |
|---|---|
| service | `xyz.openbmc_project.State.Host` |
| object | `/xyz/openbmc_project/state/host0` |
| interface | `xyz.openbmc_project.State.Host` |
| property | `CurrentHostState`（string：`Off` / `On` / `Quiesced` / `TransitionToOn` / `TransitionToOff` / `Restarting`） |
| property | `ChassisPowerTimeouts`（遷移 timeout） |
| method | `PowerOn()` / `PowerOff()` / `PowerCycle()` / `WarmReset()`（WarmReset 非所有平台都有） |
| interface | `xyz.openbmc_project.State.DecoratedHostState`，property `DecoratedState`（例如 `Ready`、`Busy:Powering On`；Redfish 拿它顯示狀態） |

Power restore policy（同 repo 的 `power-restore-policy` daemon 另起一個 service）：

| 項目 | 值 |
|---|---|
| object | `/xyz/openbmc_project/control/host0/power_restore_policy` |
| interface | `xyz.openbmc_project.Control.Power.RestorePolicy` |
| property | `Policy`（string：`AlwaysRestore` / `AlwaysOff` / `PreviousState`） |

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| phosphor-power 的 JSON config（描述各 power state 的 step 順序：GPIO 寫入、delay、依賴） | sequencer 的資料來源，隨 recipe 安裝 |
| `power-restore-policy` 的設定檔 | 預設 restore policy |
| host 狀態持久化檔（供 `PreviousState` 用） | 記錄上一次的 host 狀態 |

## 新手常問

- **狀態卡住 `TransitionToOn` 很久是什麼意思？** sequence 沒跑完：PWRGD 沒起來（硬體實際沒上電），或某個 step 的依賴（其他 GPIO 狀態）不成立。看 daemon log 與 CPLD / GPIO 實際狀態。
- **`PowerOff` 跟 GracefulShutdown 差別？** D-Bus 方法都是 `PowerOff()`；差別在前端層——graceful 先請 host 端正常關機（ACPI / IPMI 電源指令），force 直接叫 `PowerOff()`。
- **host 已經 On 還能再叫 `PowerOn()` 嗎？** 不會重跑 sequence：daemon 對重複 / 非法遷去做去重或拒絕（依實作回错误或 no-op）。
- **誰在讀 `CurrentHostState`？** 所有前端——Redfish system 的 power state、IPMI chassis status、bmcweb 的事件訂閱——都讀這一個 property，不各自維護副本。
- **要加新的電源狀態怎麼做？** 改 JSON config（新的 state / step / 依賴），重編 recipe；狀態列舉值本身在程式碼裡定義。
