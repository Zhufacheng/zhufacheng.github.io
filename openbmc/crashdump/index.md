# crashdump / coredump

OpenBMC 處理兩種「dump」：**host crashdump**（host CPU 掛掉時，BMC out-of-band 抓 host 的記憶體/CR dump）與 **BMC coredump**（BMC 自己的 daemon crash 時抓 core）。兩者都存到磁碟，事後再拉出來分析。

## 背景概念

- **host crashdump**：host CPU 發生致命錯誤（MCE / CATERR / IERR）時，BMC 趁 host 還「停在故障點」或靠 fault 前後寫入的 crash buffer，**從 host 端**讀出一段記憶體/CR（machine check 相關資料）存起來。這塊**高度依賴平台**（CPU、chipset、有沒有專用 sideband 讀取介面）。
- **MCE / CATERR / IERR**：CPU 的錯誤訊號。MCE（Machine Check Exception）是 CPU 偵測到不可回復錯誤；CATERR/IERR 是 CPU 在致命錯誤時斷言的**錯誤 pin**，BMC 透過 GPIO/CPLD 看到。
- **crash buffer**：很多平台讓 host 韌體/BIOS 在 fault 前後把 crash 資料寫到一段**已知記憶體區**，BMC 之後讀這段。
- **BMC coredump**：BMC 上某個 daemon（C/C++ 程序）segfault / 收到致命 signal 時產生的 core 映像，用來事後 debug。

## 怎麼運作

### 1. host crashdump（out-of-band，平台相關）

```
host CPU 致命 fault（MCE / CATERR / IERR）
   │
   ▼
CPU 斷言錯誤 pin（CATERR/IERR）→ CPLD latch
   │
   ▼
BMC 偵測到 pin（GPIO / CPLD / 專用介面）
   │
   ▼
BMC 讀 host crash buffer（記憶體 / MCA / CR 區）
   （sideband memory read / eSPI / 專用 crashdump 控制器）
   │
   ▼
存到 BMC 檔案系統（平台定義的路徑）
   │
   ▼
事後：工程師用 Redfish / SSH / support tarball 拉出來分析
```

重點：**BMC 在 host 可能已經無回應時仍能抓**——因為是 out-of-band 讀取。具體怎麼讀（哪個 pin、哪段記憶體、什麼介面）因平台而異，**沒有統一的上游 daemon/repo 名稱**。

### 2. BMC coredump

```
BMC daemon crash（segfault / 致命 signal）
   │
   ▼
core 被抓住（systemd-coredump / coredump handler）
   │
   ▼
phosphor-debug-collector 的 `dump` 命令把 core + log + 系統狀態打包
   │
   ▼
tarball 存到 /var/lib/phosphor-debug-collector/
```

- `phosphor-debug-collector` 提供 `dump` 命令，一鍵收集 coredump、log、設定、D-Bus 狀態等成一個 tarball，方便寄給支援/分析。
- 通常也會把「發生 crash / 抓到 dump」這件事記進 elog（見 [phosphor-logging](../phosphor-logging/)）。

## 相關專案 / daemon

- `phosphor-debug-collector`：BMC 端的 `dump` 命令（收集 coredump + 系統資料成 tarball）。
- `phosphor-logging`：elog，記錄「發生 crash / 抓到 dump」的事件。
- （host crashdump 部分：**平台相關**，通常由平台專屬的 crashdump 韌體/驅動 + CPLD 完成，沒有統一的上游 repo 名。）

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/var/lib/phosphor-debug-collector/` | `dump` 命令產出的 tarball 存放處（含 coredump） |
| host crashdump 儲存路徑 | 平台定義（常是專屬 crashdump 目錄）；依平台而異 |

## 新手常問

- **host crashdump 跟 BMC coredump 不一樣嗎？** 不一樣：host crashdump 是抓 **host CPU** 的故障資料（BMC 從 host 端 out-of-band 讀）；BMC coredump 是抓 **BMC 自己** daemon crash 的 core。
- **為什麼 BMC 能在 host 掛掉時還抓得到？** 因為是 out-of-band：BMC 用自己（獨立）的介面讀 host 記憶體/CR，不依賴 host 還活著。
- **crashdump 一定抓得到嗎？** 看平台：要有對應的錯誤 pin 偵測 + 可讀取的 crash buffer/介面。有些平台只記到「發生過 CATERR」但不含完整 memory dump。
- **BMC daemon 掛了怎麼抓 core？** `dump`（`phosphor-debug-collector`）會把 coredump 一起打包；也可看 `systemd-coredump` 的 core 存放處。
- **抓到 dump 怎麼拿出來？** 走 Redfish（下載 dump）、SSH scp，或 `dump` 命令產出的 tarball。
