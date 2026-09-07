# PLDM

PLDM（Platform Data Model，DMTF DSP0240）是管理訊息的資料模型協定，跑在 MCTP 上。它定義一組「message type」（每個 type 是一套 command/response/event），涵蓋 firmware update、FRU data、sensor data、BIOS 設定、SMBIOS 等。OpenBMC 上由 `pldmd` 作為常駐 PLDM daemon。

## 背景概念

- **PLDM Base（DSP0240）**：共同規則——訊息格式、TBT（Terminal Bus Transaction）、Request/Response、message type discovery（`RequestCommands`）。每個 PLDM message type 有自己的 Type ID。
- **Message types**（各有獨立 DMTF spec）：
  - **Firmware Update（FSP，DSP0267）**：firmware 升級協定。initiator（通常是 BMC）與 responder（目標裝置 firmware）negotiate 後分塊送 image、hash 驗證、觸發 activation。
  - **FRU Data**：裝置的 FRU 資料（part number、serial 等），作用類似 IPMI FRU 但走 MCTP/PLDM。
  - **Sensors / Telemetry**：sensor 讀值與 threshold event。
  - **BIOS**：BIOS 設定讀寫、boot option override 等（host 端用途）。
  - **SMBIOS**：SMBIOS 資料搬移。
- **Initiator / Responder**：每個 message type 有 client/server 角色。例如 firmware update：BMC 是 initiator，目標裝置是 responder。
- **Transport**：PLDM 跑在 MCTP（msg type `0x01`）；物理鏈路是 I2C/SMBus、PCIe VDM 等；對 host 端常見 PCIe VDM / eSPI。

## 怎麼運作

一般訊息流：

```
BMC（initiator）                        目標裝置（responder）
   │  RequestCommands（發現支援哪些 type）──────────────▶
   │  ◀──────────── Response：type list + version ──────
   │
   │  Get FRU / RequestSensorValues / ...（依 type）────▶
   │  ◀──────────────────── Response / Event ───────────
```

Firmware update（PLDM FSP）流程：

```
phosphor-software-manager（FSP initiator）      目標裝置 firmware（FSP responder）
   │  RequestUpdate（type、image size、hash 演算法）────▶
   │  ◀──────── RequestUpdateResp（OK、chunk size 等）──
   │  TransferFirmwareData × N（逐 chunk 送 image）────▶
   │  RequestUpdateComplete ────────────────────────────▶
   │  ◀── hash 驗證，依 activation policy 決定何時啟用
   │     （on next reset / immediate / ...）
   │  ◀── event：responder 回報 activation 完成與狀態
```

- **Initiator 端**（BMC 去 update 遠端元件）實作在 `phosphor-software-manager`：由 Redfish UpdateService 驅動，對 PLDM FSP 類目標做 negotiate → transfer → activate → 輪詢狀態；PLDM 訊息交換經 `pldmd` / `mctpd`（見 host-firmware-update 頁）。
- **Responder 端**（目標是 BMC 本身）在 `pldmd`（BMC 被 PLDM update 的場景）。
- **PLDM FRU**：`pldmd` 收裝置回報的 FRU data，在 BMC 上持久化並 expose 到 D-Bus，供 inventory / Redfish 使用。
- **host 端 PLDM**：同一套協定，transport 換成 PCIe VDM / eSPI；訊息內容與 BMC 端一致。

## 相關專案 / daemon

- `pldmd`（`openbmc/pldmd`）：PLDM daemon。實作 PLDM Base、PLDM FRU、PLDM FSP responder 等；經 `mctpd` 與 MCTP endpoint 通訊。
- `mctpd` / `libmctp`：MCTP transport 層（見 MCTP 頁）。
- `phosphor-software-manager`：FSP initiator、Redfish UpdateService 的執行後端。

## D-Bus

- `pldmd` 以 `xyz.openbmc_project.PLDM` service name expose PLDM 相關 service（物件在 `/xyz/openbmc_project/...` 下）：
  - PLDM FRU data 作為 D-Bus 物件 expose（供 inventory / Redfish 消費）；
  - FSP 相關狀態 / 進度也經 D-Bus expose。
- `phosphor-software-manager` 的 update 流程經 `xyz.openbmc_project.Software.Update` / `Software.Activation` 系 interface（activation 進度是 `Progress` property）。

## 新手常問

- **PLDM 和 IPMI 的關係？** 都是管理資料模型；IPMI 是 legacy 協定，PLDM 是較新的 DMTF 資料模型且跑在 MCTP 上。新元件（NIC、Retimer 等）多用 PLDM。
- **BMC 自己可以用 PLDM 更新嗎？** 通常不需要——BMC 自己更新走本地 file（Redfish UpdateService → `phosphor-software-manager`，不經 PLDM）。PLDM FSP 的場景是「BMC 去 update 遠端元件」。
- **PLDM FRU 資料會進 Redfish 嗎？** 會。`pldmd` 在 D-Bus 上 expose 的 PLDM FRU data 可供 inventory / Redfish 資源使用。
- **MCTP payload 只有 64 bytes，firmware update 怎麼辦？** FSP 本來就設計成分塊：`TransferFirmwareData` 逐 chunk 送，image size 與 hash 在 `RequestUpdate` 時先 declare。
- **裝置只支援 MCTP 不支援 PLDM 呢？** MCTP 只是 transport；不支援 PLDM 可以改用其他上層協定（NC-SI、vendor-defined type）或 vendor 方式。
