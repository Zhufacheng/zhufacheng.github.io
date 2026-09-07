# Host firmware update

BMC 更新 **host 端**韌體（BIOS/UEFI、CPLD、NVMe SSD、NIC…）的路徑，跟更新 BMC 自己的 image 是兩回事。主要有兩條路徑：**PLDM FSP over MCTP**（`pldmd` + `phosphor-software-manager` 驅動）與 **Redfish UpdateService**（`bmcweb`）。Redfish 通常是入口（API），PLDM FSP 是 BMC 與 host 元件之間實際交換的協定；兩者可串接。

## 背景概念

- **PLDM（Platform Data Model）**：DMTF 的平台管理協定；其韌體更新元件稱為 **PLDM FSP**（Firmware Update）。BMC 當 PLDM requester（發命令），host 端當 responder（BIOS 的 PLDM agent，或裝置內建的 FSP responder）。
- **MCTP**：PLDM 的傳輸層，典型走 SMBus/I2C；PCIe 裝置（如 NVMe）可走 PCIe VDM。
- **Component**：FSP 的更新單位，一種韌體（BIOS、NVMe…）一個 component，各有 component ID 與 version。
- **Redfish update**：`POST /redfish/v1/UpdateService/FirmwareImages` 上傳 image → `POST /redfish/v1/UpdateService` 觸發 → 產生 UpdateTask 追蹤進度。

## 怎麼運作

### 路徑 A：PLDM FSP（over MCTP）

FSP 的 command flow 分四段：

```
BMC（pldmd ⇄ phosphor-software-manager）
   │  MCTP（SMBus / PCIe VDM）
   ▼
host PLDM FSP responder（BIOS agent / 裝置）

1. discovery   GetFirmwareParameters
                → 可更新 component 清單、目前 version、支援的更新方式
2. transfer    TransferFirmwareImage / RequestImage
                → image 切塊送進裝置
3. activation  ActivateFirmwareImage
                → 裝置把 image 寫入自己的 update 區、標記 ready（passivate）
4. commit      commit / restart
                → host reboot（BIOS）或裝置 reset，正式切換
```

依序：

1. **Discovery**：`pldmd` 在 MCTP bus 上找到 PLDM FSP responder（依 EID），用 `GetFirmwareParameters` 查可更新的 component、目前 version、支援的更新方法。
2. **Transfer**：BMC 把 image 切成 chunk 送出，裝置端接收、暫存。
3. **Activation**：`ActivateFirmwareImage` 讓裝置正式把 image 寫入 update 區並標記「ready」。BIOS 的常見做法是寫入 alternate 區、等下次 reboot 生效。
4. **Commit**：裝置/BIOS 在 restart 時完成切換（BIOS = host reboot；NVMe = device commit/reset）。切換後裝置回報新 version。

host component 的 version 與 activation 狀態由 `phosphor-software-manager` 同樣記成 `Software.Version` 物件（以 `Purpose` 區分元件類型），外部介面（Redfish / IPMI）才能統一查詢每個 component 的狀態。

### 路徑 B：Redfish UpdateService（bmcweb）

1. `POST /redfish/v1/UpdateService/FirmwareImages`：上傳韌體 image，回一個 FirmwareImage resource（`@odata.id`）。
2. `POST /redfish/v1/UpdateService`：body 指定 image 的 `@odata.id` 與目標（例如 `/redfish/v1/Systems/system`），建立 UpdateTask。
3. bmcweb 把更新轉發給對應後端：更新 BMC image 走 `phosphor-software-manager` 的 BMC 路徑；更新 host component 走 PLDM FSP 路徑（路徑 A）。
4. 輪詢 UpdateTask（狀態、percent）取得進度，直到完成。

> 上傳的 image 暫存在 bmcweb 的 temp 目錄（預設 `/tmp`），實際更新由後端 daemon 執行。

## 相關專案 / daemon

- `pldmd`：PLDM daemon，PLDM over MCTP 的 command 交換（含 FSP）。
- `mctpd`：MCTP endpoint daemon，提供 MCTP 傳輸層。
- `phosphor-software-manager`：把 component 的 version / activation 狀態記成 `Software.Version` 物件；BMC image 本身也走它。
- `bmcweb`：Redfish `UpdateService` / `FirmwareImages` / UpdateTask。
- host 端：BIOS 的 PLDM FSP agent 或裝置內建 FSP responder（不在 OpenBMC 這邊）。

## D-Bus

- component 的 version 物件與 BMC image 同構：service `xyz.openbmc_project.Software.Version`、path `/xyz/openbmc_project/software/<version>`、interface `Software.Version`（`Purpose` 區分元件類型）+ `Software.Activation`（狀態機，可訂閱 `PropertiesChanged`）。
- PLDM 協定層本身沒有統一的公開 D-Bus interface 定義（隨平台實作不同），對外可見的狀態收斂在上述 version 物件上。

## 新手常問

- **跟 BMC image 更新差在哪？** BMC image = BMC 自己的 A/B rootfs，由 `phosphor-software-manager` + U-Boot rollback 管理。host 韌體存在 host/裝置端、由裝置/BIOS 自己 commit，BMC 只是上傳與觸發。
- **為什麼有兩條路徑？** Redfish 是**入口**（標準 API），PLDM FSP 是**協定**（跟 host component 說話的方式）。Redfish 觸發的 host 更新，底層通常就是走 PLDM FSP。
- **BIOS 什麼時候生效？** 通常要 host reboot（寫入 alternate 區、由 BIOS 自己的 boot 邏輯切換）。
- **rollback 誰管？** 裝置/BIOS 自己的雙 image 與開機成功判定；BMC 端不做統一的 host 韌體 rollback。
- **NVMe 也走 PLDM？** 多數 NVMe SSD 內建 FSP responder（MCTP over PCIe VDM）；部分平台由 BIOS agent 代管，視硬體而定。
