# Time

OpenBMC 的 BMC 通常沒有可靠的 RTC，所以系統時間要靠 **NTP 同步**或**手動設定**。時間相關的入口——NTP client、IPMI `Set/Get System Time`、Redfish `DateTime`——底層都匯流到同一個系統時鐘，並用 `xyz.openbmc_project.Time` 的 D-Bus 介面協調「現在這顆時鐘可不可信」。

## 背景概念

- **NTP client**：開機後持續同步（上游常用 `systemd-timesyncd`，部分平台用 `ntp` / `chrony`）；server 清單可靜態設定，也可經 D-Bus（`phosphor-network` 的 NTP 設定）改。
- **手動設時**：IPMI `Set System Time` 或 Redfish Manager `DateTime`；設完通常要讓 NTP 暫停或重新同步，避免被 NTP 拉回去。
- **時區**：系統時鐘內部一律 UTC；顯示用的 offset / 時區另存（`/etc/localtime` 那套），Redfish 用 `UtcOffset` 表達。

## 怎麼運作

```
                ┌────────────────────────────┐
   NTP server ─►│  NTP client                │
                │  (timesyncd / ntp)         │
                └──────────┬─────────────────┘
                           │ 定期同步
                           ▼
                    系統時鐘 (UTC)
                           ▲
                           │ settimeofday
        ┌──────────────────┴──────────────────┐
        │                                     │
  IPMI Set System Time                Redfish PATCH DateTime
  (phosphor-ipmi-host)                (bmcweb)
        └──────────────────┬──────────────────┘
                           │ 經 D-Bus（xyz.openbmc_project.Time
                           │ 命名空間，如 Time.Synchronize）協調
                           ▼
                    套用到系統時鐘
```

1. **開機**：NTP client 起來、向配置的 server 同步；之前存下的時間（如果有）被覆蓋。
2. **NTP 同步**：client 定期向 server 要時間、修正系統時鐘；同步狀態發在 D-Bus 上（`Time.Synchronize` 一類介面），consumer 靠它判斷「現在時間可不可信」。
3. **手動設時**：
   - IPMI：`ipmitool mc settime …`（`Set System Time`）由 `phosphor-ipmi-host` 處理，經 D-Bus 套用到系統時鐘；`Set System Time Policy` 決定之後 NTP 要不要繼續同步。
   - Redfish：PATCH `/redfish/v1/Managers/bmc` 的 `DateTime`；`/redfish/v1/DateTimeService` 管理 NTP source（`.../NTP/Sources/<id>`，可用 `ApplyTimeSynchronization`）。
4. **時區**：不影響系統時鐘（永遠 UTC），只影響顯示成什麼時區；Redfish 讀 Manager `DateTime` 會帶 offset，改 offset 就是改平台時區設定。

## 相關專案 / daemon

- `phosphor-ipmi-host`：IPMI `Set/Get System Time`、`Set System Time Policy` handler
- `bmcweb`：Redfish `DateTime` / `DateTimeService`（NTP source、`UtcOffset`）
- `phosphor-network`：NTP server 設定的 D-Bus 介面（改 server 走這）
- NTP client 本身：`systemd-timesyncd`（上游常見）或 `ntp` / `chrony`

## D-Bus

- `xyz.openbmc_project.Time.*`：NTP / 手動同步的協調介面（如 `Time.Synchronize`）——consumer 用它看同步狀態、或觸發重同步。
- NTP server 設定：在 `xyz.openbmc_project.Network`（`phosphor-network`）的 NTP 介面上。

## 新手常問

- **手動設了時間，NTP 會不會蓋掉？** 看 time policy：NTP 同步仍啟用的話，client 下次同步就把時間拉回 server 值；`Set System Time Policy` / Redfish 可指定只設一次、之後不再同步。
- **開機後還沒同步之前，時間是什麼？** 不可信（0 或上次存下的舊值）；依賴精確時間的 daemon 應該等 D-Bus 上的同步訊號再動作。
- **為什麼不直接信 RTC？** BMC 的 RTC 通常沒備援電池、精度差，所以 OpenBMC 一律以 NTP / 手動設定為準，RTC 頂多當開機暫存。
- **Redfish `DateTime` 為什麼帶 +00:00？** 系統時鐘是 UTC；offset 表示「顯示時區」，跟實際時刻無關。
