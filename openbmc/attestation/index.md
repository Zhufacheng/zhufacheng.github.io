# Attestation

Attestation（遠端驗證）是「向第三方證明：這台 BMC / host 上跑的軟體與狀態是可信的、符合預期」。做法是把「量測結果」（boot chain 各階段的 hash、韌體版本、OS image hash……）打包成一份 **attestation report**，讓 verifier 對照預期值（golden reference / signed manifest）或驗證簽章鏈（TPM quote）。

> 這是 OpenBMC 比較新、且**高度平台 / 廠商依賴**的領域：上游有 `phosphor-attestation` 一類的專案在推進，但實際部署形態（有沒有 TPM、走什麼協議、report 格式）各平台不同。本頁講通用架構。

## 背景概念

- **measured boot**：boot chain 每個階段（bootloader、kernel、initramfs……）被量測（hash 進 TPM 的 PCR 或 event log）。attestation 的「證據」來自這一步——沒有 measured boot，報告就只有「hash 對照」的強度。
- **TPM quote**：TPM 用 attestation key 對一組 PCR 值簽名，verifier 用對應公鍵驗簽、並確認 PCR 值符合預期——目前最常見的遠端驗證機制。BMC 端要 BMC 自己有 TPM 才能出真正的 quote；host 端的 TPM quote 是另一套（host 自己的 TPM）。
- **manifest 對照（弱一档）**：沒有 TPM 時，daemon 把韌體 / 映像的 hash 跟一份 signed manifest 比對，把結果報出來。強度取決於「誰量測的」——如果 BMC 本身已被篡改，它報的 hash 也不可信，所以這層通常搭配 verified boot 一起用。
- **verifier**：站在管理側（如 fleet management 的 compliance 服務）驗證 report 的一方；不是 BMC 上的元件。

## 怎麼運作

```
量測來源（BMC 上）                     verifier（管理側）
──────────────────────────           ─────────────────
measured boot 各階段 hash
phosphor-software-manager 的版本 + hash
OS image / rootfs hash
（可選）TPM PCR 值
        │
        ▼
attestation daemon（如 phosphor-attestation）
   ├─ 從 D-Bus / 檔案收集量測
   ├─ 打包成 report（hash 清單，或 TPM quote）
   └─ 經 D-Bus 曝露（bmcweb 轉成 Redfish 給管理層拉）
                                 ◄── verifier 拉取（或主動上報）
                                 驗簽（quote / manifest）
                                 對照 golden reference
                                 → pass / fail（compliance 告警）
```

1. **量測**：boot 鏈路各階段的 hash（measured boot / verified boot 的 by-product）、`phosphor-software-manager` 手上的韌體版本與映像 hash、必要時 TPM PCR。
2. **彙整**：attestation daemon 把量測收齊、打包成 report。
3. **送達**：通常掛 D-Bus、由 `bmcweb` 轉成 Redfish 端點讓 fleet 管理主動拉；也有平台做成主動上報。
4. **驗證**：verifier 驗簽章（TPM quote 的 attestation key，或 manifest 的簽章）、對照預期值，給 pass / fail。

## 相關專案 / daemon

- `phosphor-attestation`：上游的 attestation 服務（較新，各平台的完整度不一）。
- `phosphor-software-manager`：提供「現在裝的是哪個版本、映像 hash 是多少」的量測來源。
- `bmcweb`：把 report 曝露成 Redfish 給管理側。
- U-Boot / kernel（measured boot 配置）：量測的來源端。

## 新手常問

- **跟 secure boot 的關係？** secure boot 是「預防」（boot 時擋篡改）；attestation 是「證明」（事後給證據）。有 verified / measured boot，attestation 的證據才硬；沒有 TPM 時只剩 hash 對照，強度有限。
- **BMC 上的 attestation 驗的是誰？** 驗的是 **BMC 自己的** boot chain 與韌體。host 的 attestation（host TPM、host measured boot）是 host 端的事，BMC 最多當觀察者。
- **沒有 TPM 能不能做？** 能做但弱：hash 跟 signed manifest 比對。前提要信任「量測的那個 daemon 本身沒被改」——所以通常要求先有 verified boot。
- **report 走什麼協議？** 依平台：常見是 Redfish 端點被 fleet 管理拉取；TPM quote 的形式要跟 verifier 的实现对得上。沒有統一強制標準。
- **跟 SEL / 事件記錄有什麼差別？** SEL 記「發生了什麼」（event）；attestation 答「現在的狀態是否符合預期」（state verification），通常用在 compliance / supply-chain 檢查。
