# KVM over IP

KVM over IP 是「遠端帶外 console」：把 host 的 VGA 畫面（video）透過網路送到 client，並把 client 的鍵盤（有些平台含滑鼠）送進 host，用來在 host 還沒起 OS、或 OS 掛掉時做 setup / debug。跟 [SOL](../sol/) 的差別：SOL 只搬 serial 文字流；KVM 搬的是 framebuffer 等級的畫面 + 鍵盤。

> 注意：上游 OpenBMC 社群**沒有**標準的 KVM daemon——各平台 / 廠商自建。本頁講通用架構與資料流，不綁定特定 repo。

## 背景概念

- **video capture path**：host 的 VGA 輸出（LVDS / VGA 訊號）沒辦法直接被 BMC 的 CPU 讀到，需要一顆「video capture」硬體（capture chip 或 USB capture 裝置）把畫面轉成 BMC 能取的 frame buffer。capture 裝置的種類與接法依平台而異。
- **keyboard injection**：BMC 把「鍵盤事件」變成 host 認識的 input。常見做法是 BMC 模擬一個 USB HID keyboard（透過 USB gadget 接到 host 的 USB host controller）；有些平台走 PS/2 或 eSPI。
- **session**：同一時間通常只有一個 KVM session（console 是獨占資源）；新 session 會取代舊的。

## 怎麼運作

```
host VGA output
      │
      ▼
video capture 硬體（capture chip / USB capture）
      │  frame（raw 或 MJPEG）
      ▼
KVM service（平台自製）
      ├─ video：encode → 網路串流（HTTP/WebSocket、VNC/RFB 或私有協議）
      │                              │
      │                              ▼
      │                    KVM client（瀏覽器 / VNC viewer）
      │
      └─ keyboard：client 事件 → HID gadget（USB 進 host）
                                    │
                                    ▼
                              host input stack（BIOS / OS 都收得到）
```

1. **開 session**：client 先通過認證——KVM 服務通常對接 `phosphor-user-manager` 的帳號與 privilege（一般只對 Administrator / Operator 等級的帳號開 console）。
2. **video 路徑**：capture 硬體產生 frame → KVM 服務取 frame → 按需 encode（低解析度可 raw，高解析度多壓成 MJPEG 縮流量）→ 串流到 client。frame rate 與解析度取決於 capture 硬體與 encode 方式。
3. **keyboard 路徑**：client 的 key event → KVM 服務翻譯成 USB HID report → 經 HID gadget 送進 host。host 端看起來就是一顆普通鍵盤——BIOS / UEFI 的 setup 畫面、OS 的 login 都收得到。
4. **關 session**：client 斷線或逾時 → 釋放 console。

## 相關專案 / daemon

- 平台自製的 KVM / video service（名稱各平台不同；video capture 裝置的 driver 通常也一併在平台 layer 提供）。
- `phosphor-user-manager`：session 的認證與授權來源。
- 若 KVM 走 Web UI，通常由 `bmcweb` 或平台的 Web 前端認證後轉給 KVM 服務。

## 新手常問

- **host 完全沒電時看得到畫面嗎？** 看不到。video 要 host 至少到 BIOS 出圖；完全斷電時只有 BMC 自己的 console（BMC 端的 serial / SSH）。
- **SOL 跟 KVM 怎麼選？** 要看 serial console（OS 的 getty、BIOS 的 serial port）就用 SOL——省頻寬、字元級；要圖形（UEFI setup 的 GUI、X、VGA 上的 debug 訊息）就用 KVM。
- **為什麼各平台解析度 / 流暢度差很多？** 取決於 capture 硬體的 frame rate、位元率，以及 encode 方式（raw 對網路壓力大，MJPEG 好很多但多一層延遲）。
- **為什麼上游沒有標準 daemon？** video capture 硬體與 host 端的 keyboard 注入路徑都是平台硬體綁定的，社群沒有統一模組；各家在 vendor layer 各自實作。
- **安全上要注意什麼？** console 是最高權限的存取——務必確認 KVM session 綁定 user-manager 的 privilege，且只走加密通道（HTTPS / WSS 或 VPN）。
