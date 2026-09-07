# Video / USB

BMC 上有兩種「呈現給 host」的能力：擷取 host 的 VGA 影像做 KVM over IP（遠端圖形 console）；以及 USB device 模擬／USB virtual media——把存在 BMC 上的 image 以 USB CD／隨身碟的形式呈現給 host，用來開機或灌 driver。

## 背景概念

- **KVM over IP**：在 BMC 上實現的「video + 鍵盤 + 滑鼠」遠端 console；video 來源是 host 的 VGA 輸出訊號。
- **VGA capture engine**：現代 BMC SoC（如 Aspeed AST2600）內建 video engine，硬體擷取 host 的 VGA 訊號成 BMC OS 可讀取的 frame buffer。
- **USB device 模擬 (VHCI)**：userspace 實現一個「virtual USB host controller」（VHCI，Virtual Host Controller Interface 協定），向 host 模擬出 USB 裝置（如 mass storage），host 列舉時當成一般 USB 裝置。
- **Virtual media**：ISO image 存在 BMC，透過 USB 裝置（或 virtual CD）呈現給 host，host 可從它開機或裝 driver，不必走網路。

## 怎麼運作

### Video（KVM over IP）

```
host VGA output
   │
   ▼
BMC SoC video engine (VGA capture, 硬體)
   │
   ▼
frame buffer (BMC OS 可讀：fbdev / V4L2 device)
   │
   ▼
KVM video 路徑：抓 frame → JPEG/MJPEG encode → IP 串流 (HTTP)
   │
   ▼
遠端 client (KVM web UI / browser)
```

- 鍵盤／滑鼠透過 virtual channel（virtual PS/2 或 USB 裝置，視平台）注入 host。
- 上游 OpenBMC 沒有統一的「KVM video daemon」repo：較新的 OpenBMC 把 KVM over IP 功能整合進 `bmcweb`（web server，由 video service 取畫面、轉發鍵盤滑鼠），各平台也可自製 KVM daemon——資料路徑相同：capture → encode → IP 串流。

### USB（virtual media）

```
ISO image (存在 BMC 磁碟)
   │
   ▼
以 block device 形式呈現給 userspace (例如經 NBD)
   │
   ▼
userspace virtual USB host controller (VHCI 協定)
   │  模擬 USB mass storage 裝置
   ▼
host USB bus
   │  host 列舉成 USB CD-ROM / 隨身碟
   ▼
host 從它開機 / 裝 driver (不必 PXE)
```

- 具體 daemon 實現因平台／vendor 而異（上游沒有統一 repo），但概念相同：image 被呈現為「USB 上的 block 裝置」。
- 給 host 用的 USB port 也可以是實體 USB hub（BMC SoC 的 USB hub controller），用於 USB redirection——遠端 session 接入 host 的 USB 裝置。

## 相關專案 / daemon

- `bmcweb` — Redfish／web server，新版已整合 KVM over IP（video + 鍵盤滑鼠）
- kernel：Aspeed video engine driver（VGA capture → frame buffer）、USB host controller driver
- 平台特定的 KVM daemon／USB virtual media daemon — 實現因平台而異，無統一上游 repo

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| fbdev / V4L2 video device（名稱視平台） | VGA capture 的 frame buffer |
| virtual media image 目錄（平台自定） | ISO image 存放 |

## 新手常問

- **BMC 怎麼「看得到」host 螢幕？** SoC 的 video engine 硬體擷取 host 的 VGA 訊號，不是軟體；支援的解析度／frame rate 受 video engine 限制，host 切到 video engine 不支援的解析度或圖形模式時畫面可能凍結或花屏。
- **Virtual media 一定要走 USB 嗎？** 不一定：也可以以 virtual CD 等形式呈現；USB 裝置模擬（VHCI）是最常見的形態，且 host 端不需要特殊支援。
- **KVM over IP 跟 serial console (SOL) 差在哪？** KVM 是影像（圖形 console）；SOL 是文字 serial（見 Host interface 頁）。crash 時往往只剩 serial 有輸出，兩者都要會用。
- **鍵盤滑鼠怎麼注入？** 透過 virtual PS/2 或 USB 裝置 channel（視平台），KVM web UI 的輸入由 BMC 轉發給 host。
