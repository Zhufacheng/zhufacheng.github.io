# Secure boot

「Secure boot」在 OpenBMC 語境裡有兩層意思，本頁分開講：

1. **BMC 自己的 verified boot**：BMC 的 bootloader（U-Boot）在 boot 前驗證 kernel / 映像的簽章，驗不過就不 boot。
2. **host 端 UEFI Secure Boot 的監控**：host 的 UEFI secure boot 狀態（PK / KEK / db / dbx、mode）由 BMC 讀取並對外回報（Redfish / IPMI）——BMC 不管理 host 的 key，只當 observer。

## 背景概念

- **簽章映像（signed image）**：映像（FIT image、kernel 等）用私鍵簽名（典型：RSA 簽 hash，hash 用 SHA-256）。驗證方持有對應公鍵（或公鍵的 hash），驗「映像沒被改 + 簽者是可信的」。
- **root of trust（信任根）**：簽章鏈的最終錨點，兩種常見形式：
  - **silicon eFuse / OTP**：SoC 的 boot ROM 用燒在 eFuse 裡的公鍵 hash 驗證第一階 bootloader——最硬、不可改。
  - **burned-in key**：公鍵（或 hash）編譯進 U-Boot 映像本身——信任根變成「U-Boot 映像本身沒被改」，強度依賴前面有沒有 eFuse 層。
- **FIT image**：U-Boot 的映像格式，可把 kernel + FDT（+ 其他 component）打包，每個 component 獨立簽章 / 驗證。
- **UEFI Secure Boot 變數**：host 端的 PK（Platform Key）、KEK（Key Exchange Key）、`db`（可信簽名名單）、`dbx`（被拒簽名名單）；加上 mode = Enabled / Disabled。

## 怎麼運作

### BMC verified boot

```
SoC boot ROM
   │  （有 eFuse key：驗證第一階 bootloader 簽章；沒 eFuse：直接信任映像）
   ▼
U-Boot
   │  讀 kernel 映像（FIT）
   │  用內建的公鍵 hash 驗證簽章 + hash
   ├─ 失敗 → 停止 boot（不跳 kernel）
   └─ 通過 → 跳 kernel
                │
                ▼
           kernel + rootfs
```

- 上游 OpenBMC 預設的驗證範圍主要是 **kernel + FDT**（打成 signed FIT，RSA + SHA-256 是常見組合）。
- **rootfs 是否納入簽章視平台**：有的平台把 rootfs 也放進 FIT、有的另行處理；沒納入的話，rootfs 的完整性依賴「U-Boot 之後的路徑沒被改」這個假設。
- 簽章用的私鍵不進映像；build 流程（Yocto recipe）在打包時簽，U-Boot 端只帶公鍵（hash）。
- 跟韌體更新（`phosphor-software-manager`）是兩道關：verified boot 管「boot 時映像的完整性」，update 路徑管「寫進 flash 的映像是否可信」（通常也驗簽章）。

### host UEFI Secure Boot 監控

- host 的 UEFI 把 secure boot 狀態放在 NVRAM 變數（PK / KEK / db / dbx + mode）。
- BMC 經 host 介面（常見是 eSPI，或 IPMI OEM command）讀這些狀態，由 `bmcweb` 曝露成 Redfish 的 SecureBoot 資源，供管理層查詢 / 告警。
- 部分平台會把「host secure boot 被關掉」當成 security event 寫進 SEL。

### IOPMP 與平台安全

- **IOPMP**（I/O and Memory Protection）：ARM 生態的硬體單位，限制「哪些 I/O master / CPU 可以存取哪些記憶體區間」——擋不受信任的 DMA 裝置讀到 protected memory，常跟 TEE / secure world 搭配。
- 跟 secure boot 是不同層：secure boot 保證「跑起來的 code 沒被改」，IOPMP 保證「即使有不信任的 DMA 行為，記憶體還是隔離的」。平台安全架構裡兩者常一起出現，但互不取代。

## 相關專案 / daemon

- `u-boot`（aspeed 平台的 `u-boot-aspeed` recipe）：BMC 端的驗證點。
- `phosphor-software-manager`：韌體更新映像的簽章 / 驗證（update 路徑，不是 boot 路徑）。
- `bmcweb`：host secure boot 狀態的 Redfish 回報。
- `phosphor-logging`：security event（例如 host 被關 secure boot）的 SEL 記錄。

## 新手常問

- **驗證失敗會怎樣？** U-Boot 停在 verification 步驟、不跳 kernel——BMC 起不來（或停在 U-Boot console，視平台）。這是 by design 的 fail-closed。
- **公鍵放哪？** 通常編譯進 U-Boot（公鍵 hash 形式）；支援 eFuse 的 SoC 可把 trust anchor 放 eFuse，強度更高。
- **BMC 能「修」host 的 secure boot 嗎？** 不能。host 的 PK / KEK / db / dbx 由 host 的 UEFI 管理；BMC 只讀、只回報。
- **verified boot 跟 measured boot / attestation 的差別？** verified = boot 時擋篡改（fail-closed）；measured = 把每個階段的 hash 量進 TPM / event log，之後可被遠端驗證（見 [Attestation](../attestation/)）。
- **簽章方案各家一樣嗎？** 不一定。FIT + RSA + SHA-256 是上游常見組合，但 key 層級、eFuse 用法、rootfs 是否納入驗證都依平台而異。
