# MCTP

MCTP（Management Component Transport Protocol，DMTF DSP0236）是管理訊息域用的 transport 協定：讓管理元件（BMC、host、device 的 firmware controller）在不同物理鏈路（I2C/SMBus、PCIe VDM、KCS 等）上有**統一的訊息通道**，上層協定（PLDM、NC-SI 等）跑在它上面。OpenBMC 用 `libmctp` 與 `mctpd` 實作 MCTP stack。

## 背景概念

- **EID（Endpoint ID）**：MCTP 的位址。每個 MCTP endpoint 有一個 1-byte EID（有效 1–254，255 為 broadcast）。MCTP 訊息 header 帶 src/dest EID 與 msg type；EID 位址化對上層透明，換 transport 也不用改。
- **Msg type**：1 byte，區分上層協定：`0x00` NC-SI、`0x01` PLDM、`0x7F` vendor-defined。MCTP 本身不 parse payload，只按 type 分發給對應上層。
- **物理 transport**：MCTP 可載於 I2C/SMBus、PCIe VDM（Vendor-Defined Message）、KCS（對 host）、USB 等，各有定義好的 framing；位址化與訊息格式跨 transport 一致。
- **Discovery（EID discovery）**：MCTP controller（通常是 BMC）掃出 bus 上的 endpoint 並指派 EID（discovery request/response、Set/Get Endpoint ID、DiscoveryComplete）。I2C/SMBus 上 SMBus 位址由 EID 派生（常用 0x07 起，broadcast 位址 0x71）；PCIe 上在 VDM enumeration 時發現。
- **64 bytes**：MCTP packet 的 payload 上限（預設 MTU）是 64 bytes，較大的上層資料（如 firmware image）由上層協定自行分塊傳送。

## 怎麼運作

```
 Host ────(KCS)──────────────────┐
 PCIe device ──(PCIe VDM)───────┤
 NIC / Retimer / … ─(I2C/SMBus)─▶  mctpd（libmctp）
                                    │ 驗 header、依 dest EID 路由
                                    │ 送到本機 → 依 msg type 分發
                                    ▼
                                0x00 → NC-SI 處理（見 NCSI 頁）
                                0x01 → pldmd（PLDM，見 PLDM 頁）
                                0x7F → vendor 處理
```

1. **Bus 初始化**：`mctpd` 依硬體描述（哪些 I2C bus 是 MCTP bus、PCIe root、KCS channel）開啟各 transport。
2. **Discovery**：對每個 bus 跑 EID discovery——發 discovery notice/request，endpoint 回應、BMC 指派 EID；每個發現的 endpoint 建一個 peer 物件（EID、transport、bus 資訊）。
3. **訊息路由**：MCTP packet 從某 transport 進來 → `mctpd` 驗 header、依 dest EID 路由；若送到本機，看 msg type 分發給對應上層協定（例如 `pldmd`）。
4. **發送**：上層 daemon（如 `pldmd`）要傳訊息時，透過 D-Bus interface 把 MCTP message 交給 `mctpd`，由它依 dest EID 選對 transport 送出。
5. **承載 PLDM / NC-SI**：PLDM（DSP0240）與 NC-SI 就是 MCTP 上兩個 msg type（`0x01` / `0x00`）的上層協定；MCTP 只保證位址化與投遞。

## 相關專案 / daemon

- `libmctp`（`openbmc/libmctp`）：MCTP protocol stack 函式庫（endpoint、EID 管理、message dispatch），同時提供 `mctpd` daemon。
- `mctpd`：常駐 daemon，管 transport 與 endpoint，並在 D-Bus 上提供 MCTP message 送收 interface 給其他 daemon。
- `pldmd`：PLDM over MCTP 的 consumer（見 PLDM 頁）。
- NC-SI（NCSI 類）：MCTP msg type `0x00` 的 consumer（見 NCSI 頁）。

## D-Bus

- `mctpd` 以 `xyz.openbmc_project.MCTP` service name expose MCTP service（物件在 `/xyz/openbmc_project/...` 下）：
  - 每個已發現的 endpoint 一個 peer 物件（EID、transport、bus 資訊）；
  - 讓其他 daemon 送 MCTP message、註冊 message handler（依 msg type 收分發）的 interface。
- `pldmd` 等上層 daemon 都是這組 interface 的 consumer。

## 新手常問

- **為什麼不直接在 I2C 上談？** MCTP 把「位址化 + msg type + transport」統一：上層（PLDM/NC-SI）不關心理論上 endpoint 在 I2C 還是 PCIe，同一套 API。
- **MCTP payload 只有 64 bytes 怎麼辦？** 這是 SMBus framing 的限制；大資料由上層協定分塊（例如 PLDM FSP firmware update 逐 chunk transfer）。
- **EID 怎麼指派？** BMC 當 MCTP controller，在 discovery 過程指派；endpoint 端通常是 dynamic EID，重啟後重跑 discovery。
- **MCTP 和 SMBus/I2C 是一回事嗎？** 不是。I2C/SMBus 是物理層；MCTP 是在它上面的訊息協定（有特定 framing），同一套 MCTP 也可跑 PCIe VDM。
- **host 端怎麼接 MCTP？** 常見走 PCIe VDM 或 KCS；BMC 端由 `mctpd` 對應 transport 處理。
