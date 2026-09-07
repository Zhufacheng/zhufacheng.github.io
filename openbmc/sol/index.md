# SOL (Serial-over-LAN)

SOL（Serial-over-LAN）是 IPMI 標準定義的「serial console over network」：把 host CPU 的 serial port（UART）資料流，透過 BMC 用 RMCP+ 上的 SOL payload 搬到遠端 client（典型工具：`ipmitool sol activate`）。host 端不需要任何 agent——BMC 只是橋。

## 背景概念

- **RMCP+ payload**：IPMI over LAN（IPMI 2.0）在 RMCP+ session 上跑多類 payload；一般 IPMI command 是一類，**SOL 是另一類 payload（payload type `0x08`）**——走同一個已認證的 session，但資料路徑獨立。
- **host serial**：host CPU 的 serial0（UART）實體接到 BMC 上的一個 tty。host 端（UEFI、OS 的 getty）把 console 掛在這個 serial 上，BMC 端讀寫那個 tty 就能看到 / 送出 console 內容。
- **SOL config parameters**：baud rate、character length、parity、stop bits、flow control、retry count / timeout——要對齊 host serial 的設定，否則資料會錯。

## 怎麼運作

### 控制路徑（啟用 / 設定）

SOL 的控制走 IPMI 的 SOL command（NetFn `Transport`）：

| Command | 作用 |
|---|---|
| Get SOL Payload Info | 查 BMC 支援的 SOL 參數 |
| Set SOL Configuration Parameters | 設 baud / parity / retry 等 |
| Get SOL State | 查有沒有 active session |
| Activate SOL | 開啟 console 橋 |
| Deactivate SOL | 關閉 |

```
client (ipmitool sol activate)
   │  1. 建立 RMCP+ session（user 認證、privilege 綁定）
   │  2. IPMI: Activate SOL
   ▼
BMC (phosphor-ipmi-net)
   │  3. attach host serial tty
   │  4. 開 SOL payload 通道
   │
   ▼  5. 雙向資料流（SOL payload，type 0x08）
   ├─→ host UART（client 打出去的 key）
   └─← host UART（console 輸出）
```

1. **認證**：client 先建立 RMCP+ session——user 認證走 `phosphor-user-manager`（D-Bus 上的帳號與 privilege）。
2. **activate**：IPMI Activate SOL command → `phosphor-ipmi-net` attach host 的 serial tty、開啟 SOL payload 通道。同一時間只有一個 active SOL session（第二個 activate 會被拒）。
3. **資料流**：tty 進來的 byte → 包成 SOL payload → RMCP+ 送出；client 的 key → SOL payload → 寫進 tty → host。SOL payload 內建 retry（retry count / timeout 可配），網路抖動時字元可重傳。
4. **deactivate**：client 送出 deactivate（或 session 斷掉）→ 釋放 tty。

### 跟其他 console 的關係

- SOL 只搬「serial port 上的字元」——host 必須真的把 console 放 serial 上（UEFI 的 serial redirect、OS 的 `console=ttyS*` / getty）才看得到東西。
- 要圖形畫面（UEFI setup、X）→ 用 [KVM over IP](../kvm-over-ip/)。
- Redfish 端：`bmcweb` 提供 SerialInterface 資源與 SolActivate / SolDeactivate action，背後呼叫 `phosphor-ipmi-net` 在 D-Bus 上曝露的 SOL 管理介面（activate / deactivate 與參數 property）。

## 相關專案 / daemon

- `phosphor-ipmi-net`：IPMI over LAN（RMCP+）實作，SOL payload 與 SOL command 都在這裡。
- `phosphor-user-manager`：session 的 user 認證與 privilege。
- `bmcweb`：Redfish SOL action 的入口。

## D-Bus

SOL session 本身走 RMCP+ payload，不走 D-Bus method；但「啟用狀態與設定參數」會由 `phosphor-ipmi-net` 同步到 D-Bus（activate / deactivate 方法與 baud 等 property，`xyz.openbmc_project.Ipmi` 命名空間），供 `bmcweb` 與本機工具查詢使用。

## 新手常問

- **`ipmitool sol activate` 連上去卻沒畫面？** 先確認 host 端 console 在 serial 上（UEFI 的 serial redirect、OS 的 getty）；再確認 SOL 的 baud / parity 跟 host 一致。
- **SOL 很卡、會掉字？** RMCP+ 是可靠傳輸，但 SOL 的 retry 參數（retry count / timeout）預設偏保守；網路高延遲時字元流會變慢。提高 baud 不會變快——瓶頸在 RMCP+ 的封裝與網路，不在 UART。
- **多個 client 可以同時看嗎？** 不行，同一時間只有一個 active session；第二個 activate 通常被拒（SOL 已 active）。
- **SOL 需要 host 端跑 agent 嗎？** 不需要。BMC 端是純橋；host 端只要 firmware / OS 把 console 掛在 serial 上。
- **跟 KVM over IP 的關係？** 互補：SOL = serial 字元流（輕、早期 boot 就有）；KVM = 圖形 console（重、要 capture 硬體）。debug 時常兩個一起開。
