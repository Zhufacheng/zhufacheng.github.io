# Host interface

host 與 BMC 之間需要好幾種連結：IPMI 指令通道、sideband 訊號（button／中斷）、serial console。傳統 LPC 時代 KCS／BT／SMBus 都掛在 LPC 上；eSPI 時代同一批協定改由 eSPI channel 承載；serial 則用來做 console 擷取與 debug。本頁說明各 channel 做什麼、host 端的一筆指令怎麼到達 BMC。

## 背景概念

- **KCS (Keyboard Controller Style)**：mailbox 式寄存器組——host 把指令寫進寄存器、BMC 處理後寫回結果；最常見的 IPMI channel。
- **BT (BMC Target, BT-C)**：另一種與 KCS 同類的 mailbox 式寄存器介面，用於部分老平台（host kernel driver `ipmi_btc`）。
- **SSIF**：走 SMBus 的 IPMI（host kernel driver `ipmi-ssif`）；新平台可透過 SMBus-over-eSPI-OOB，不需要實體 SMBus 線。
- **LPC**：legacy 低 pin count 匯流排，KCS 時代承載 KCS IO port，現多被 eSPI 取代。
- **Serial (RS-232)**：host console UART ↔ BMC serial port；host console 擷取（SOL）與 debug 用。

## 怎麼運作

### 各 channel

1. **KCS**：host 端 kernel `ipmi-kcs` driver（MMIO 或 IO port 寄存器組）→ `/dev/ipmi0` → openipmi (libipmi)。現代平台的 PCH 把 KCS mailbox 透過 eSPI Peripheral channel 呈現給 host——host 看到的仍是「KCS」，只是物理層變成 eSPI。
2. **BT (BT-C)**：IO port 寄存器組（LPC 時代），host kernel `ipmi-btc`；legacy 平台使用。
3. **SSIF**：SMBus 兩線（或 eSPI OOB）；BMC 端 IPMI 以 SMBus slave 身分監聽。
4. **Serial**：host console UART 實體接 BMC；BMC 擷取 host console 串流（crash log、debug），`ipmitool sol`（Serial-over-LAN）讓遠端 terminal 接到這個 console。

### host 端一筆指令怎麼到 BMC（以 KCS + IPMI 為例）

```
host:  app (ipmitool -I open ... / vendor tool)
          │
          ▼
       openipmi (libipmi)
          │
          ▼
       kernel ipmi_msghandler ── ipmi-kcs: 寫入 KCS mailbox 寄存器
                                   (MMIO；現代平台走 eSPI Peripheral)
          │ netlink (host 端 IPMI handler 在 userspace 時)
          ▼
      ══════ 物理鏈路 (KCS / SMBus / eSPI) ══════
          ▼
BMC:   phosphor-host-ipmi (IPMI command 處理)
          │
          ▼
       D-Bus dispatch → 目標 app
       (phosphor-power / SEL / sensor / ...)
          │
          ▼
       response 原路返回 → host app
```

### 用途對照

| Channel | 用途 |
|---|---|
| KCS | IPMI 指令通道（OS 層 local 管理，最常見） |
| BT (BT-C) | legacy IPMI 寄存器介面 |
| SSIF | 走 SMBus 的 IPMI（新平台可走 eSPI OOB） |
| eSPI | 現代 host 連結（承載 KCS/SSIF/VW/OOB，見 SPI / eSPI 頁） |
| Serial | host console 擷取、debug、SOL |

## 相關專案 / daemon

- host kernel：`ipmi-kcs`（`ipmi_kcs`）、`ipmi-btc`（`ipmi_btc`）、`ipmi-ssif`、openipmi（`ipmi_msghandler`）
- BMC 端：`phosphor-host-ipmi` — IPMI 指令處理 + D-Bus dispatch（含 SOL、IPMI watchdog，見 Watchdog 頁）
- 目標 app：`phosphor-power`（power／reset）、SEL daemon、sensor daemon 等——IPMI 指令最終作用在這些 daemon 上

## 新手常問

- **host 上的 `ipmitool -I open` 是用 KCS 嗎？** 用 openipmi local interface（`/dev/ipmi0`）；底層 channel 由平台決定，現代 server 多數是 KCS（over eSPI）。
- **KCS 跟 SSIF 差在哪？** KCS 是寄存器 mailbox、host 單向發起；SSIF 是 SMBus 訊息、BMC 可用 SMBALERT# 主動中斷 host；新平台的 SSIF 可走 eSPI OOB 承載。
- **SOL 為什麼需要 serial？** SOL 是把 host 的 serial console 串流轉送出去；host UART 實體接在 BMC 上，BMC 把它封裝後送出。
- **eSPI 完全取代 LPC 了嗎？** 新平台是；eSPI Peripheral channel 相容 legacy LPC 的 IO/CFG 語意，host 端 KCS driver 基本透明。
