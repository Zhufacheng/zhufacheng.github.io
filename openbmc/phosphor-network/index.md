# phosphor-network

`phosphor-network` 是 OpenBMC 的網路管理 daemon：把 BMC 的網路介面（`eth0`…）、IP（IPv4/IPv6、static/DHCP）、VLAN、MAC、DNS、NTP 設定全部暴露成 D-Bus 物件。改 D-Bus property = 改網路設定——daemon 收到變更就立即套用到介面並持久化。

## 背景概念

- **物件模型**：每個網路介面 = 一個 D-Bus 物件（`/xyz/openbmc_project/network/eth0`）；IP 位址、VLAN 都是它的**子物件**。
- **event-driven**：daemon 不 poll；D-Bus property 被 set 就觸發重設定，介面狀態變動（netlink 事件）也回寫 D-Bus。
- **IP 來源模式**：`Origin` 決定 IP 怎麼來的：`None` / `Static`（手動設 `Address` / `Netmask` / `Gateway`）/ `DHCP`（跑 DHCP client）/ `LinkLocal`。

## 怎麼運作

### 物件樹

```
/xyz/openbmc_project/network/              root / 全域設定（DNS 等）
└── eth0/          xyz.openbmc_project.Network.EthernetInterface
    │               （MACAddress、MTU …）
    ├── ipv4/      xyz.openbmc_project.Network.IP.Interface
    │              （Mode=IPv4、Origin=None/Static/DHCP/LinkLocal）
    │   └── 1/     xyz.openbmc_project.Network.IP
    │              （Address、Netmask、Gateway、State …）
    ├── ipv6/      同上（IPv6 版）
    └── vlan2/     xyz.openbmc_project.Network.VLAN（Id=2）
                    （VLAN 介面下同樣可以有 ipv4/ipv6 子物件）
```

- 一個介面可以有**多個 IP**（`ipv4/1`、`ipv4/2`…），各自是獨立子物件。
- VLAN：在介面下建 `vlan<N>` 子物件（`Id` = VLAN id），實體上對應一個子介面（`eth0.2` 之類）。

### 改 property → 重設定

```
client（bmcweb / busctl / 其他 daemon）
   │  D-Bus Set
   │  （例：/network/eth0/ipv4 的 Origin 設成 DHCP；
   │    或在 /network/eth0/ipv4/1 上設 Address/Netmask/Gateway）
   ▼
phosphor-network property setter
   ├─ 立即套用：netlink 改介面（IP、MAC、MTU、VLAN、DHCP client 起/停）
   ├─ 寫持久化設定（開機後恢復）
   └─ 狀態變動再發回 D-Bus（PropertiesChanged / 子物件增刪）
```

典型操作對照：

| 想做什麼 | D-Bus 操作 |
|---|---|
| 設 static IP | `/network/eth0/ipv4` 的 `Origin` → `Static`；建/改 `/network/eth0/ipv4/1` 的 `Address` / `Netmask` / `Gateway` |
| 切 DHCP | `Origin` → `DHCP` |
| 改 MAC | 設 `eth0` 的 `MACAddress` |
| 建 VLAN | 在 `eth0` 下建 `vlan123` 子物件（`Id` = 123） |
| 關 IPv6 | `/network/eth0/ipv6` 的 `Origin` → `None` |

- DNS / NTP：也走同一棵物件樹的設定（root 物件上的介面），改完 daemon 更新對應的系統設定檔；NTP server 換掉後 NTP client 重新同步。
- 開機恢復：daemon 讀持久化設定，重建 D-Bus 物件並套用。

## 相關專案 / daemon

- `phosphor-network`（本頁主角，D-Bus 服務 `xyz.openbmc_project.Network`）
- `bmcweb`（Redfish `NetworkInterfaces` / `Ethernets` / NIC 設定；Redfish PATCH 轉成 D-Bus Set）
- `phosphor-ipmi-host`（IPMI LAN Channel 命令，讀寫同一組 D-Bus 物件）
- `phosphor-dbus-interfaces`（`Network/*.interface` 介面定義）
- NTP client 本身（`systemd-timesyncd` / `ntp`，看平台選哪個）

## D-Bus

- service：`xyz.openbmc_project.Network`
- 根路徑：`/xyz/openbmc_project/network`（root 物件 + 全域設定如 DNS）
- 主要介面：
  - `Network.EthernetInterface`：`MACAddress`、`MTU` 等介面屬性
  - `Network.IP.Interface`：`Mode`（IPv4/IPv6）、`Origin`（None/Static/DHCP/LinkLocal）
  - `Network.IP`：`Address`、`Netmask`、`Gateway`、`State`（一個 IP 位址）
  - `Network.VLAN`：`Id`

## 新手常問

- **為什麼 IP 是子物件、不是介面的 property？** 一個介面可以有數個 IP（static 設的、DHCP 拿到的、link-local），物件模型自然支援多組，consumer 也可以用 match 盯「任何 IP 變化」。
- **DHCP 拿到 IP 後 D-Bus 會怎樣？** `eth0/ipv4` 的 `Origin` 顯示 DHCP，daemon 建出 `ipv4/<n>` 子物件放 DHCP 拿到的 `Address` / `Gateway`。
- **改設定要重開機嗎？** 不要，setter 立即生效 + 持久化。
- **bmcweb 改 IP 走什麼路徑？** Redfish PATCH → `bmcweb` → D-Bus Set（`phosphor-network`）→ netlink 套用；沒有繞過 D-Bus 的另一條路。
- **NTP server 在哪改？** 同一棵樹（root 物件的 NTP 設定）；Redfish 端對應 DateTime Service 的 NTP sources。
