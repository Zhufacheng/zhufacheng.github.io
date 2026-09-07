# NCSI / PNCC

NCSI（Network Controller Sideband Interface）是 network controller（NC，含 PHY / port mux）的 out-of-band 管理 sideband 介面：經 sideband channel 查詢 link / PHY 狀態、force link 狀態、並選擇網路埠 traffic 走哪一邊（host 或 BMC）。PNCC 是這條 sideband command channel 的稱謂（Intel NCSI 以 LPC/KCS 為經典 transport；較新的設計把同一組命令搬到 MCTP，即 NC-SI）。這是 OpenBMC 上較小的 feature，BMC 端主要是 `phosphor-ncsi`（NCSI client）。

## 背景概念

- **NC（Network Controller）**：host / BMC 與 PHY 之間網路路徑上的控制器（例如 NIC 的 sideband controller，或專用的 mux / PHY 管理晶片）。
- **Sideband channel**：與 data path 分開的控制 channel。Intel NCSI 定義一組命令集，經典物理 transport 是 LPC/KCS；DMTF 的 NC-SI spec 把類似功能集包在 MCTP 上跑（msg type `0x00`）。
- **典型 NCSI commands**：`GetNCVersion` / `GetNCCapabilities`（能力查詢）、`GetLinkStatus` / `SetLinkStatus`（查 / force link）、`GetPHYStatus` / `SetPHYStatus`（PHY 查 / force）、`Get/Set PHY Auto-Negotiation`、`Select Port`（選預設 port：host 或 BMC）、`GetNCInfo` / `GetVendorInfo` 等。
- **Port selection**：網路埠由 host 與 BMC 共用時，用 NCSI 命令決定 traffic 走哪邊——常見情境：OOB 管理時切到 BMC port，正常運作切回 host port。

## 怎麼運作

```
BMC（phosphor-ncsi，NCSI client）
   │  NCSI command（例：GetLinkStatus / Select Port）
   │
   ▼  sideband channel（經典：KCS/LPC；較新：MCTP）
NC（network controller / PHY / mux）
   │
   ▼  response（link state、PHY status、port state...）
BMC 把狀態 expose 到 D-Bus → Redfish / 監控使用
```

1. BMC 端的 `phosphor-ncsi` 以 NCSI client 角色起來，經 sideband channel 與 NC 通訊。
2. 可定時（或 on demand）查詢 link / PHY 狀態並 expose 到 D-Bus——host 掛掉時 BMC 仍能看到網路狀態（OOB monitoring）。
3. port selection 情境：由 BMC（或上層管理介面）發 Select Port 命令，把網路埠在 host 與 BMC 之間切換。
4. 若平台走 MCTP transport（NC-SI over MCTP），訊息交換經 `mctpd`（msg type `0x00`），其餘流程與 MCTP/PLDM 頁相同。

## 相關專案 / daemon

- `phosphor-ncsi`（`openbmc/phosphor-ncsi`）：OpenBMC 端的 NCSI client daemon，實作 NCSI command 流程，把 NC 狀態 expose 到 D-Bus。
- （MCTP transport 時）`mctpd` / `libmctp`。

## D-Bus

`phosphor-ncsi` 把 NC / link 狀態（link state、port state 等）expose 在 D-Bus 的 `/xyz/openbmc_project/...` 下，供 Redfish 等其他管理介面消費。（具體物件 path 與 interface 名以該 daemon 目前版本為準；此 feature 部署面不如其他管理介面廣。）

## 新手常問

- **NCSI 和 NC-SI 差在哪？** NCSI 是 Intel 原版的 sideband spec（經典 LPC/KCS transport）；NC-SI 是 DMTF 標準化版本、跑在 MCTP 上。命令功能相近（查 link/PHY、port selection）。
- **沒有 host/BMC 共用 port 的機器需要 NCSI 嗎？** 不一定要有；但 OOB link monitoring（BMC 看 PHY 狀態）與是否共用 port 無關，也走 NCSI 命令。
- **NCSI 和 Redfish 的關係？** NCSI 是對 NC 的 out-of-band channel；BMC 端查到的狀態 expose 到 D-Bus 後，Redfish 可以呈現。
- **這 feature 普及嗎？** 相對小——只有有 sideband 可管理 NC 的 platform 才需要；多數 OpenBMC 機器不會用到 NCSI 路徑。
