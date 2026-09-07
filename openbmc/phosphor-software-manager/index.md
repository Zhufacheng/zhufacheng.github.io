# phosphor-software-manager

`phosphor-software-manager` 是 OpenBMC 的 **BMC 映像版本管理** daemon：把每一份已安裝的 BMC image 當成一個 D-Bus 物件（`Software.Version`），負責 image 的驗證、寫入 standby 分割區（staging）、activation（Active/Ready）、A/B 與 rollback。它管的是 **BMC 自己的 rootfs**；host 端（BIOS/CPLD/NVMe…）的元件韌體更新走另一條路徑（host firmware update 條目）。

## 背景概念

- **`Software.Version` 物件**：每個已安裝的 BMC version 都是 `/xyz/openbmc_project/software/` 下的一個 D-Bus 物件，物件名就是 version 字串，帶 `Version`、`Purpose`、`Updateable` 等 property。
- **Activation 狀態**（`xyz.openbmc_project.Software.Activation.ActivationStates` 列舉）：`Staged` → `Ready` → `Active`，中間有 `Activating` / `Deactivating`，失敗為 `Failed`。
  - `Staged`：image 已驗證並寫入磁碟、尚未設為可開機。
  - `Ready`：image 已就緒在 standby 分割區，等待 activation。
  - `Active`：目前執行中（或已指定開機）的 image。
- **A/B 分割區**：rootfs 分成兩片（常見名 `rootfs_A` / `rootfs_B`），各放一份 image；任何時刻一份在跑、另一份待命，兩片對稱。
- **`current` / `active` / `standby`**：software 路徑下的三個固定別名物件——`current` = 目前執行的 image、`active` = 標記為 Active 的 version、`standby` = 另一片的 image。`current` 與 `active` 通常指向同一份 version。
- **Rollback**：新 image 開機失敗時，由 U-Boot 的 `bootcount` 機制自動改開舊分割區；開機完成後 daemon 再把失敗的 version 標記成 `Failed`。

## 怎麼運作

```
image（上傳 / 寫入）
   │
   ▼
ApplyUpdate
   ├─ 驗證（size、signature…）
   └─ 寫入 standby 分割區
   │
   ▼
Staged
   │
   ▼
SetRequestedActivation(Ready)
   │
   ▼
Ready
   │
   ▼
SetRequestedActivation(Active)
   │   更新 U-Boot env（boot 目標切到另一片）
   ▼
reboot
   │
   ├─ 開機成功 ──► 新 image 變 Active，舊 image 回到 Ready
   └─ 開機失敗（bootcount 達門檻）
        └─► U-Boot 切回舊分割區開機
             → 開機後 daemon 標記新 version 為 Failed
```

依序：

1. **開機註冊 version**：daemon 讀取目前 rootfs 與另一片的 version 字串，建立對應的 `Software.Version` 物件（含 `current` / `active` / `standby` 別名）。
2. **`ApplyUpdate`**：caller（bmcweb 的 Redfish 上傳、IPMI、或 script）傳 image 位置（本機路徑或 URL）；daemon 驗證後寫入 standby 分割區，成功則該 version 狀態為 `Staged`。
3. **Activation**：寫 `RequestedActivation` property（`SetRequestedActivation`）。設 `Ready` = 標記該 version 就緒可啟用；設 `Active` = daemon 更新 U-Boot 的 boot 選擇（env）。**此時新 image 還沒在跑**，要 reboot 才切換。
4. **Reboot 與切換**：重開機後 U-Boot 依 env 開另一片。新 image 開機成功，OS 端寫回 boot 成功標記，daemon 更新狀態（新 version `Active`、舊 version `Ready`）。
5. **Rollback**：新 image 開機失敗（U-Boot 沒收到成功標記、bootcount 達門檻）→ U-Boot 自動改開舊片；開機後 daemon 偵測到執行的仍是舊 version，把新 version 標 `Failed`。

> 各更新入口（Redfish / IPMI / script）最後都是對這個 daemon 的 `ApplyUpdate` + `SetRequestedActivation`，只是包装不同。

## 相關專案 / daemon

- `phosphor-software-manager`：本條目主角，提供 version 物件、activation 狀態機與 image 寫入。
- `bmcweb`：Redfish `UpdateService`（upload + trigger），收到後轉發給本 daemon。
- `phosphor-bmc-code-mgmt`：**build 端**的 image 打包 recipe（rootfs → 可開機 image，含簽章步驟），不是 runtime daemon。
- `pldmd` + MCTP：host 端元件韌體更新路徑（不經本 daemon 的 A/B rootfs 機制），見 host firmware update 條目。
- U-Boot：bootloader，負責分割區選擇、signature 驗證、bootcount rollback。

## D-Bus

- **well-known name**：`xyz.openbmc_project.Software.Version`。
- **object path**：`/xyz/openbmc_project/software/<version>`（version 字串為物件名）；另有 `current`、`active`、`standby` 固定別名物件。根節點是 Object Manager（`xyz.openbmc_project.Object.Manager`）。
- **interfaces**：
  - `xyz.openbmc_project.Software.Version`：`Version`（string）、`Purpose`（string，BMC image 為 `BMC`）、`Updateable`（bool）。
  - `xyz.openbmc_project.Software.Activation`：`Activation`（目前狀態）、`RequestedActivation`（可寫，值 `None` / `Ready` / `Active`）；新版另有 `BootNext`（指定下次開哪個 A/B slot）。
- **methods**：`ApplyUpdate`（參數含 image 位置——檔案路徑或 URL，可附 version 字串）、`EraseVersion`（移除某 version）。
- **signals**：標準 `PropertiesChanged`，可訂閱狀態機轉移。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/tmp/<uuid>`（bmcweb 暫存） | Redfish 上傳後 image 的暫存處，接著交 `ApplyUpdate` |
| A/B rootfs 分割區（常見 `rootfs_A` / `rootfs_B`） | BMC image 的兩個 slot |
| `ubootenv` 分割區 | U-Boot env：boot slot 選擇、bootcount、成功標記 |

具體分割區名稱由平台 partition table 決定；重點在「A/B 兩片 + env」這套機制。

## 新手常問

- **跟 host firmware update 差在哪？** 本 daemon 管 **BMC 自己**的 rootfs（A/B + U-Boot rollback）；host 端 BIOS/CPLD/NVMe 等走 PLDM FSP / Redfish 元件更新（`pldmd`、`bmcweb`），韌體存在 host/裝置端、commit 方式也不同。
- **更新完成就生效嗎？** 不是。`SetRequestedActivation(Active)` 之後只是 boot 選擇改了，要 reboot 才開新 image。
- **rollback 是誰做的？** 「切換」動作是 U-Boot（bootcount + 成功標記）做的；daemon 事後標記失敗 version、整理狀態。
- **`current` 跟 `active` 為什麼要有兩個？** `current` = 正在跑的；`active` = 被標記 Active 的。activation 完成、reboot 前，兩者可以短暫不一致。
- **一次可以放幾份 version？** 標準 A/B layout 最多兩份（current + standby）；`EraseVersion` 用來清除某 version 的物件與記錄。
