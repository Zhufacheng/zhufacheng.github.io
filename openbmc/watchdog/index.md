# Watchdog

Watchdog 是「不喂就 reset」的機制：BMC 提供一個給 host 用的 watchdog timer——host（BIOS 或 OS）沒在时限內 refresh（kick）它，BMC 就執行設定好的動作（hard reset、power cycle...）。BMC 自身另有一個 self-watchdog，BMC 軟體掛掉時自動 reset BMC。

## 背景概念

- **Host watchdog**：IPMI watchdog；host 定期用 IPMI kick，逾時後 BMC 執行動作。
- **kick**：refresh watchdog——把剩餘時間重置回初始值。
- **Expire action**：逾時時執行的行為：hard reset、power cycle、power down、NMI、none 等。
- **BMC self-watchdog**：BMC SoC 上的硬體 wdt + userspace 餵狗；保護的是「BMC 本身」（與 host watchdog 獨立）。

## 怎麼運作

### Host watchdog（IPMI → D-Bus）

1. **設定**：host 端（BIOS，或 Linux kernel `ipmi-wdt` driver）用 IPMI 設定 watchdog：
   - `Set Watchdog Timer` (0x22)：timer use、逾時動作、mode（countdown/countup）、初始計數
   - `Get Watchdog Timer` (0x23)：查狀態／剩餘時間
   - `Stop Watchdog Timer` (0x25)：停止
2. **定期 kick**：開機前 BIOS 定期 kick；OS 開起來後由 kernel `ipmi-wdt` 接手餵。
3. **IPMI ↔ D-Bus 對應**：`phosphor-host-ipmi` 收到 IPMI watchdog 指令，對應到 D-Bus watchdog 物件。
4. **`phosphor-watchdog` daemon**（通用 D-Bus watchdog）：
   - interface `xyz.openbmc_project.State.Watchdog`：property `enabled`（bool）；method `Interval`（get/set，ms）、`TimeRemaining`（get/set，ms——**set 就是 kick**）。
   - D-Bus service name 與 object path 由啟動參數指定（上游 help 範例為 `/xyz/openbmc_project/state/watchdog/host0`；部分平台改用 `/xyz/openbmc_project/watchdog/host0` 一類路徑）。
   - **timeout → 動作**：daemon 啟動預先設定的 systemd target（action → unit 對應，如 hard reset／power cycle 的 target）；沒對應時執行 fallback action（間隔也可設）。
   - **postcode 自動延長**：`--watch_postcodes` 盯 host 開機 postcode（`/xyz/openbmc_project/state/boot/raw0`）變化時自動延長剩餘時間——覆蓋 BIOS 開機期間 host 還沒接手 kick 的空窗。

```
host (BIOS / kernel ipmi-wdt)
   │ IPMI 0x22/0x23/0x25 (走 KCS 或 SSIF)
   ▼
phosphor-host-ipmi
   │ D-Bus: xyz.openbmc_project.State.Watchdog
   ▼
phosphor-watchdog
   ├─ enabled / Interval / TimeRemaining
   ├─ host kick → TimeRemaining 重置
   ├─ timeout → systemd target (hard reset / power cycle)
   └─ (可選) postcode 變化 → 自動延長
```

### BMC self-watchdog

- kernel wdt driver（BMC SoC 的 watchdog timer，`/dev/watchdog`）+ userspace 定期餵狗；BMC 主流程掛掉無法餵時，硬體自動 reset BMC。
- 有些平台另有獨立 supervision 機制（與 kernel wdt 分開），監控主要 daemon 的活性。

## 相關專案 / daemon

- `phosphor-watchdog` — D-Bus watchdog daemon（service name、object path 由啟動參數指定）
- `phosphor-host-ipmi` — IPMI watchdog 指令（0x22/0x23/0x25）↔ D-Bus watchdog 對應
- host 端：Linux kernel `ipmi-wdt`（host 的 `/dev/watchdog`）— OS 的 watchdog 介面，底下就是 IPMI 到 BMC
- power 控制 daemon（`phosphor-power` 等）— watchdog 的 action target 最終由它觸發 host reset／power cycle

## D-Bus

- service name：`phosphor-watchdog` 啟動時以 command-line 參數指定
- object path：啟動時指定，例如 `/xyz/openbmc_project/state/watchdog/host0`
- interface `xyz.openbmc_project.State.Watchdog`：`enabled`（bool）、`Interval`（ms）、`TimeRemaining`（ms，set ＝ kick）
- `/xyz/openbmc_project/state/boot/raw0`（`xyz.openbmc_project.State.Boot.Raw`）— host 開機狀態／postcode，供 watchdog 自動延長

## 新手常問

- **host 完全沒有 kick 會怎樣？** 初始計數耗盡後 BMC 執行逾時動作（hard reset／power cycle）——這正是 watchdog 要達成的「自動救回」；若 host 刻意不用 watchdog，要先 `Stop Watchdog`。
- **countdown 與 countup 的差別？** countdown 是常見形態：時間從初始值往下減、歸零逾時；countup 則是時間往上加、超過上限逾時。
- **BIOS 和 OS 用同一顆 watchdog 嗎？** IPMI 規格上每個 BMC 一顆 watchdog：BIOS 先設定，開機過程 OS 接手——所以需要 postcode 自動延長機制。
- **跟 host 上的 `/dev/watchdog` 有什麼關係？** host 的 `/dev/watchdog`（`ipmi-wdt`）是給 OS 的介面，底下仍是 IPMI 到 BMC；BMC 自己的 `/dev/watchdog` 是 BMC SoC 的硬體 wdt，跟 host 無關。
- **逾時動作怎麼配？** `phosphor-watchdog` 啟動參數指定各 action 對應的 systemd unit（`--action_target`）與 fallback action（`--fallback_action`），例如 hard reset 指向 reset target unit。
