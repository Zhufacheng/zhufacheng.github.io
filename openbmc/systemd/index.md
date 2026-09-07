# systemd

OpenBMC 用 systemd 當 init（PID 1）與 service manager：所有 daemon 都跑在 systemd unit 上，開機順序、掛掉重啟、log 都交給它管。OpenBMC 的 daemon unit 幾乎一律 `Type=dbus`——「服務已啟動」的判定標準是「拿到 D-Bus well-known name」，「服務掛掉」也是用「name 消失」來判定，所以它跟 D-Bus 那一層是緊耦合的。

## 背景概念

- **unit**：systemd 管理的最小單位（`.service`、`.target`、`.socket`…）。OpenBMC 主要用 `.service`。
- **`Type=dbus` + `BusName=`**：unit 要等 process 在 system bus 上拿到 `BusName=` 指定的 well-known name 才算 STARTED；unit 還活著但 name 掉了（daemon 掛掉或主動 release name），systemd 判定 unit **failed**。
- **`Restart=always`**：任何方式 exit 都重啟；`RestartSec=` 控制重啟間隔。`[Unit]` 段的 `StartLimitIntervalSec` / `StartLimitBurst` 限制短時間內的重啟次數，避免無限重啟迴圈。
- **相依與順序**：`Requires=` / `Wants=`（拉起相依）、`After=` / `Before=`（順序）、`PartOf=`（parent 被 stop 時跟著 stop）。
- **target**：OpenBMC 預設開機目標是 `multi-user.target`（`default.target` 指到它）。
- **journal**：systemd log 系統，用 `journalctl` 查。

## 怎麼運作

### 開機與監控

```
kernel boot → systemd (PID 1)
  ├─ 讀取 unit 檔 (/usr/lib/systemd/system、/etc/systemd/system)
  ├─ default.target → multi-user.target
  │
  └─ 各 daemon unit，例如：
       [Unit]
       Description=Foo daemon
       After=...
       [Service]
       Type=dbus
       BusName=xyz.openbmc_project.Foo
       ExecStart=/usr/bin/foo
       Restart=always
       RestartSec=5
       [Install]
       WantedBy=multi-user.target
       │
       ├─ systemd 執行 ExecStart
       ├─ process request_name(BusName) 成功 → unit STARTED
       ├─ daemon 掛掉 / name 消失 → unit failed
       │     └─ Restart=always → 隔 RestartSec 重試
       │        （受 StartLimitIntervalSec/StartLimitBurst 限制，
       │           超過就進 failed 不再重啟）
       └─ systemctl stop → 正常 STOPPED
```

### enable（讓服務隨開機啟動）

- **image 建置期**：recipe 把 `.service` 裝進 unit 目錄，並由 Yocto 的 `SYSTEMD_SERVICE` + `SYSTEMD_AUTO_ENABLE` 機制在打包 rootfs 時建好 `multi-user.target.wants` symlink。
- **執行期**：`systemctl enable/disable <unit>`（寫 `/etc/systemd/system/` 的 symlink）；`systemctl start/stop` 只影響本次執行。

### 常用操作

```
systemctl status <unit>                            # 狀態 + 最近 log
systemctl list-units --type=service --state=failed # 找 failed 的服務
systemctl restart <unit>                           # 重啟
systemctl start|stop <unit>                        # 手動起停
systemctl reset-failed <unit>                      # 清掉 failed/start-limit 狀態
journalctl -u <unit> -f                            # 追這個服務的 log
journalctl -u <unit> --since "5 min ago"
systemctl cat <unit>                               # unit 內容（含 drop-in）
systemctl edit <unit>                              # 做 drop-in override
```

## 相關專案 / daemon

- `systemd` — init 本身（Yocto `systemd` recipe；OpenBMC 的 systemd 版本已支援 `BusName=`）。
- 各 OpenBMC daemon repo — `.service` unit 檔在 repo 內定義、由 recipe 安裝；改 unit 就是改 repo。
- OE-core 的 `systemd` bbclass + `systemd-systemctl-native` — image 建置期的 unit enable 機制。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/usr/lib/systemd/system/` | package 安裝的 unit 檔 |
| `/etc/systemd/system/` | 本機 override、drop-in、enable 的 symlink |
| `/etc/systemd/system/multi-user.target.wants/` | 被 enable 的 service symlink |
| `/run/log/journal/`、`/var/log/journal/` | journal（執行期通常只在 `/run`，重啟會清） |

## 新手常問

- **為什麼用 `Type=dbus` 而不是 `Type=simple`？** `Type=dbus` 下「啟動成功」=「拿到 bus name」，name 與服務狀態對得上；daemon 一掛 name 必然消失，systemd 才能可靠判定 failed 並觸發 `Restart=`。
- **服務一直 failed 重啟怎麼辦？** `systemctl status <unit>` 看 exit 原因；若被 start-limit 擋住不再重啟，先 `systemctl reset-failed <unit>` 再處理。
- **不重灌 image 怎麼改服務參數？** `systemctl edit <unit>` 做 drop-in override（`/etc/systemd/system/<unit>.d/*.conf`）；要永久生效還是改 repo 重建 image。
- **開機順序誰先誰後？** unit 的 `After=` / `Before=` 決定順序。實務上 OpenBMC 各 daemon 多靠 D-Bus 等彼此（等不到就訂閱事件），所以順序要求沒有传统 init 那麼死，但真要等某個服務就先起來就加 `After=`。
- **可以用 `Type=notify` 嗎？** 可以（daemon 發 `sd_notify` 回報準備完成），但 OpenBMC 生態主流是 `Type=dbus`，新增 daemon 建議跟著走。
