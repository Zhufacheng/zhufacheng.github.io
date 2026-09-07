# BMC boot chain

BMC 自己的開機鏈：**SoC boot ROM → U-Boot → Linux kernel → rootfs（systemd）**。rootfs 採 A/B 兩片分割區 layout，U-Boot 用 `bootcount` 機制選擇開機分割區——這正是 `phosphor-software-manager` 的 A/B 更新與自動 rollback 能成立的底層機制。

## 背景概念

- **A/B 分割區**：rootfs 切成兩片完整分割區（常見名 `rootfs_A` / `rootfs_B`），各可放一份完整 BMC image（kernel + rootfs）。任何時刻一片在跑、另一片待命。
- **`bootcount` 機制**：每個 A/B slot 有「開機失敗計數」與「開機成功標記」。U-Boot 每次選一片開機；若開出去後沒收到成功標記就回到 U-Boot，該 slot 計數 +1；計數達門檻就改開另一片。
- **成功標記（boot success）**：OS 開機成功後，OS 端把成功標記寫回 U-Boot env，U-Boot 才知道「這片是好的」，計數歸零。
- **Image 簽章**：BMC image 在 build 時簽章；U-Boot 開機前驗證 signature，驗證失敗該片不開。

## 怎麼運作

```
SoC boot ROM（SoC 內建）
  └─► U-Boot（自 boot media 載入：SPI NOR flash / eMMC）
        ├─ 1. 讀 env（ubootenv）：目前 boot slot、各 slot 的 bootcount / 成功標記
        ├─ 2. 驗證 image signature（失敗 → 該片不開）
        ├─ 3. 被選 slot 未標記成功 且 bootcount ≥ 門檻
        │       └─► 改開另一片（即「rollback」）
        └─ 4. 載入並開機所選分割區的 kernel + rootfs
              └─► systemd → 各 service（含 phosphor-software-manager）
                    └─► 開機成功後回寫成功標記到 env
                         （該 slot 標記 good、計數歸零）
```

依序：

1. **上電**：SoC ROM 從 boot media（SPI NOR flash 或 eMMC）載入 U-Boot。boot media 的 partition table 大致分成 U-Boot（含 SPL）、`ubootenv`、A/B 兩片 rootfs。
2. **選分割區**：U-Boot 讀 env——目前 boot 目標是哪個 slot、各 slot 的 bootcount 與成功標記。
3. **簽章驗證**：驗證所選分割區的 image；失敗不開（有另一片可 fallback）。
4. **開機**：該 slot 若失敗過多次（bootcount 達門檻且未標記 good），改開另一片；否則照常開機。
5. **成功確認**：OS 開機成功後，OpenBMC 端 service（通常與 software 更新流程一併實作）把成功標記寫回 env；U-Boot 下次讀到就把該 slot 標 good、清計數。

### 跟 phosphor-software-manager 的接點

- `phosphor-software-manager` 把新 image 寫進 standby 片、再 `SetRequestedActivation(Active)`，實質動作是**更新 U-Boot env 的 boot 選擇**（指向另一片）。daemon 不直接碰 kernel/rootfs 開機流程，真正切換由 U-Boot 完成。
- reboot 後若新 image 開機失敗，U-Boot 依上述機制自動開回舊片；開機後 daemon 偵測到「執行的仍是舊 version」，把新 version 標 `Failed`。
- 一句話：**daemon 管狀態機（誰 Active / Ready），U-Boot 管實際切換與 rollback**，兩邊合作完成 A/B 更新。

> 具體 env 變數名、分割區名、計數門檻各平台不同（U-Boot porting 與 partition table 決定）；機制（A/B + bootcount + 成功標記）是一致的。

## 相關專案 / daemon

- U-Boot：bootloader（上游 U-Boot + 平台 porting），負責分割區選擇、簽章驗證、bootcount。
- `phosphor-bmc-code-mgmt`：**build 端**的 image 打包 recipe（rootfs → 可開機 image，含簽章步驟）。
- `phosphor-software-manager`：runtime 的 version 狀態機、activation 時寫 env、rollback 後的標記。
- systemd：rootfs 端 init，拉起各 service。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| SoC boot ROM | 第一級 boot（SoC 內建、不可更新） |
| U-Boot（含 SPL）分割區 | bootloader 本體 |
| `ubootenv` 分割區 | U-Boot env：boot slot 選擇、bootcount、成功標記 |
| A/B rootfs 分割區（如 `rootfs_A` / `rootfs_B`） | BMC image 的兩個 slot |

具體名稱以平台 partition table 為準；重點是「U-Boot + ubootenv + A/B rootfs」這套結構。

## 新手常問

- **rollback 是誰主動做的？** U-Boot。daemon 不發「rollback」指令，只在事後標記狀態。
- **怎麼判定「開機失敗」？** 開出去後在限定時間內沒收到成功標記（回到 U-Boot），該 slot 的 bootcount +1；達設定門檻後不再被選，直到該片被重新標記 good。
- **簽章驗證失敗會怎樣？** U-Boot 拒絕開該片；若另一片可用則改開另一片，否則系統開不起來。
- **A/B 兩片會一直堆舊 version 嗎？** 不會。一片被 activation 後，另一片就是 standby（`Ready`）；下次更新覆蓋 standby 片。兩片永遠是「current + standby」。
- **跟 host BIOS 的雙 image 差在哪？** BMC 的 A/B rollback 在 U-Boot 層閉環（bootcount），不依賴 OS；host BIOS 的雙 image 由 BIOS 自己的 boot 邏輯管理，BMC 只能觀察。
