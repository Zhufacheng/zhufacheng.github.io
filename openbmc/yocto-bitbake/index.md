# Yocto / bitbake

OpenBMC 用 Yocto Project（build engine：bitbake）建置整套 BMC 韌體：rootfs（systemd + 所有 daemon）、U-Boot、device tree，最後組出可燒錄的 firmware image。理解「layer → recipe → task」這三層結構，就夠你擴充 OpenBMC（加 daemon、改別人 recipe、客製 image）。

## 背景概念

- **bitbake**：build engine。parse 所有 recipe、產生 task 相依圖、執行（含 sstate cache 重用）。
- **layer**：一組 recipe + 設定（一個 git repo）。OpenBMC 常見層級：
  - `meta-openbmc` — 在 `openbmc` repo 內：核心 image、machine 定義、核心 recipe。
  - `meta-phosphor` — OpenBMC 專案層：`phosphor-*`、`sdbusplus` 等絕大多數 recipe。
  - `meta-aspeed` — SoC vendor 層。
  - vendor / machine 層 — 特定 board 的層：machine conf、硬體配置、額外 recipe。
- **recipe（`.bb`）**：怎麼建置一個 package（抓 source、patch、compile、install、相依）。`.bbappend` 檔可在不動原 recipe 的情况下增量修改它。
- **machine（`.conf`）**：board 定義（`MACHINE`），決定硬體、kernel、DTB、image 內容。
- **image recipe**：`obmc-phosphor-image` 是 OpenBMC 的核心 image；各 machine 的 image recipe 通常 `inherit` 它、再補 `RDEPENDS` 組成完整 image。

## 怎麼運作

### build flow

```
openbmc/  (openbmc/openbmc repo)
└─ ./setup <machine>
     ├─ 抓需要的 layer
     └─ 產生 build/ 目錄 (conf/、bblayers.conf、local.conf)
          │
          ▼
cd build && bitbake obmc-phosphor-image
     ├─ parse 所有 layer 的 .bb / .bbappend
     │    (meta-openbmc、meta-phosphor、meta-aspeed、vendor 層)
     ├─ 解 task 相依圖 (DEPENDS 建置期、RDEPENDS 執行期)
     ├─ 重用 sstate-cache (沒變的 task 直接跳過)
     ├─ 各 recipe 的 task 鏈：
     │    do_fetch → do_unpack → do_patch
     │    → do_configure → do_compile → do_install
     └─ 組 image (rootfs、U-Boot、DTB、…)
          → tmp/deploy/images/<machine>/  (rootfs 與 firmware image)
```

### 一個典型 recipe（示意）

```bitbake
SUMMARY = "Foo daemon"
LICENSE = "Apache-2.0"

SRC_URI = "git://github.com/<org>/foo.git;protocol=https;branch=main"
SRCREV = "..."

inherit cmake
DEPENDS = "sdbusplus"

SYSTEMD_SERVICE:${PN} = "foo.service"
SYSTEMD_AUTO_ENABLE = "enable"

do_install:append() {
    install -d ${D}${systemd_unitdir}/system
    install -m 0644 ${S}/foo.service ${D}${systemd_unitdir}/system/
}
```

`SYSTEMD_SERVICE` + `SYSTEMD_AUTO_ENABLE` 讓 unit 在 image 打包時就被 enable（見 systemd 那篇）。

### 常用指令

```
bitbake <recipe>                        # 建置該 package
bitbake -c cleanall <recipe>            # 清掉重build
bitbake -e | grep <var>                 # 看變數最終值
bitbake-layers show-recipes '*foo*'     # 找 recipe 在哪个 layer
devtool modify <recipe>                 # 把 source 拉進 workspace 改（產出 bbappend）
devtool reset <recipe>                  # 收起來
```

### 加一個 daemon 進 image（典型步驟）

1. 在 vendor 層寫新 `.bb`（或先 `devtool modify` 現有 recipe 再改），把 binary + `.service` 裝進去。
2. 在 image recipe 加 `RDEPENDS:append = " foo"`（或 `IMAGE_INSTALL:append`）。
3. `bitbake obmc-phosphor-image`（或該 machine 的 image recipe）。

### 改別人的 recipe

- 優先用 vendor 層的 `.bbappend`：append `SRC_URI`、加 patch、覆寫 function（`do_compile:append` 之類）。
- 同名 recipe 出現在多個 layer 時，取 `BBFILE_PRIORITY` 較高者——vendor 層通常設比較高。

## 相關專案 / daemon

- `openbmc`（openbmc/openbmc）— 主 repo（含 meta-openbmc）；`setup` 腳本是 build 入口。
- `meta-phosphor` — OpenBMC 專案層。
- `meta-aspeed` — SoC 層。
- bitbake / OE-core — build engine 與核心 layer（外部依賴，不在 openbmc repo 內）。
- 各 `phosphor-*` daemon repo — recipe 抓的 source；recipe 本身住在 meta-phosphor / meta-openbmc。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `build/conf/bblayers.conf` | 啟用的 layer 清單 |
| `build/conf/local.conf` | build 設定（DISTRO、MACHINE、classes…） |
| `conf/machine/<machine>.conf` | machine 定義（在 layer 內） |
| `tmp/work/` | 各 recipe 的 build 目錄 |
| `tmp/deploy/images/<machine>/` | 產出的 image |
| `sstate-cache/` | task 結果 cache（第二次 build 快的原因） |
| `downloads/` | source / tarball cache |

## 新手常問

- **第一次 build 為什麼這麼久？** 所有 task 從零跑（含 cross toolchain 本身）；之後的 build 重用 `sstate-cache`，沒改過的 recipe 幾乎瞬間完成。
- **改了 recipe 會自動重 build 嗎？** 會——改 `.bb` / `.bbappend` 後 bitbake 偵測到 hash 變化會重跑受影響的 task；若發現沒生效，`bitbake -c cleanall <recipe>` 清掉舊狀態。
- **recipe 到底在哪個 layer？** `bitbake-layers show-recipes <name>`；同名在不同 layer 時 `BBFILE_PRIORITY` 高的贏。
- **想 debug 一個不維護的 recipe？** `devtool modify <recipe>` 把 source 拉到 workspace 改、build、測試，之後 `devtool reset <recipe>` 把修改固化成 bbappend。
- **產出在哪？** `tmp/deploy/images/<machine>/`，內容視 machine 而異，通常含 rootfs、U-Boot、DTB 與可燒錄的完整 firmware image。
