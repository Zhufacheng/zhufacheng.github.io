# I2C / I3C

BMC 上大多數被動感測器（溫度、電壓、PSU、fan tach、EEPROM...）都掛在 I2C（或其受限 profile SMBus）匯流排上。Linux 內核 I2C subsystem 把每個 controller channel 暴露成 `/dev/i2c-N` 字元裝置，OpenBMC 的 sensor daemon 與 entity-manager 就是從這些 bus 讀值、再 publish 到 D-Bus。I3C 是新一代高速繼任標準。

## 背景概念

- **I2C / SMBus**：SDA/SCL 兩線、7-bit 位址、clock stretching；SMBus 是 I2C 的受限子集（10/100 kHz、PEC），溫度／電壓／PSU 感測器多為 SMBus 形態。
- **`/dev/i2c-N`**：kernel `i2c-dev` module 為每個 I2C adapter（controller channel、mux 子通道）建立字元裝置；userspace 開啟後以 `i2c_smbus` ioctl 做 raw read/write（SMBus 協定）。
- **I2C mux**：單個 controller channel 透過 mux chip（PCA954x 系列）分出多條子 bus；kernel 裡每個 mux channel 是獨立 adapter，各有自己的 `/dev/i2c-N` 編號。
- **`new_device`**：用 sysfs 把 driver 動態 bind 到某 bus 某位址上的 device（用於放不進 device tree、或 runtime 才加上去的 device）。
- **I3C**：I2C 的 MIPI 高速繼任（12.5 Mbps，high-speed mode 32/64 Mbps），向下相容 I2C；I3C device 用 ENTDAA 取得動態位址，可用 IBI（In-Band Interrupt）帶內中斷。

## 怎麼運作

1. **adapter 建立**：device tree → I2C controller driver（BMC SoC，如 Aspeed I2C）→ 註冊 adapter `i2c-N` → `i2c-dev` 建立 `/dev/i2c-N`（udev 指派節點）。
2. **driver 綁定**，兩條路：
   - 靜態：device tree 已列 device（compatible + 位址），開機 probe 對應 driver（hwmon、eeprom...）。
   - 動態：
     ```
     i2cdetect -y 7                          # 看 0x50 是否有回應
     echo "eeprom 0x50" > /sys/bus/i2c/devices/i2c-7/new_device
     # 反綁：
     echo "eeprom 0x50" > /sys/bus/i2c/devices/i2c-7/delete_device
     ```
3. **mux**：mux 的子 channel 各拿新 adapter 編號，sysfs 上可見 child adapter 掛在 mux device 下；部分平台另把 mux 的 channel 選擇節點暴露在 `/dev/i2c-mux/...`，讓 userspace 直接切換／選取通道（例如讀取 mux 後方的感測器前先選 channel）。

```
controller channel i2c-7
  ├─ 直連 devices (eeprom, temp ...)
  └─ pca9548 (mux)
       ├─ channel 0 → adapter i2c-8 (psu, eeprom ...)
       └─ channel 1 → adapter i2c-9 (temp sensor ...)
```

4. **OpenBMC 軟體怎麼用**：
   - kernel driver 註冊 hwmon/eeprom → sysfs；sensor daemon（`phosphor-hwmon-sensor` 等）讀 hwmon 後 publish 到 D-Bus。
   - raw SMBus 存取（PSU 讀數、EEPROM）：開啟 `/dev/i2c-N` + `I2C_SMBUS` ioctl 做 byte/word read——`dbus-sensors` 裡的 I2C 存取 helper 就是這個模式。
   - `entity-manager` 在 JSON 中描述各板卡的 I2C device（I2CDevice：Bus + Address），sensor/hwmon daemon 照描述決定要拉起哪些 device。

```
I2C/SMBus device (EEPROM / temp / PSU / fan)
   │ SDA/SCL
   ▼
I2C controller (BMC SoC) ── /dev/i2c-N
   │
   ├─ kernel driver probe → hwmon / eeprom sysfs
   │      │
   │      ▼
   │   sensor daemon (phosphor-hwmon-sensor ...)
   │      │
   │      ▼
   │   /xyz/openbmc_project/sensors/...  (D-Bus)
   │
   └─ raw I2C_SMBUS ioctl (PSU / EEPROM)
          │
          ▼
      dbus-sensors 類 daemon → D-Bus
```

## 相關專案 / daemon

- kernel：I2C core + `i2c-dev`（module `i2c-dev`）、i2c-mux core 與 mux chip driver（PCA954x 系列）
- I3C：kernel `drivers/i3c` subsystem（Linux 6.16，2024 底起合併），master controller driver 包含 `i3c-master-cdns`（Cadence）、`ast2600-i3c-master`（Aspeed AST2600）、`dw-i3c-master`（DesignWare）等
- `entity-manager` — 硬體描述（I2CDevice 的 Bus/Address）
- `dbus-sensors` 家族：`phosphor-hwmon-sensor`、`phosphor-psu-manager` 等 — hwmon/sysfs 讀取 + raw SMBus 存取
- `i2c-tools` — `i2cdetect` / `i2cget` / `i2cset` / `i2cdump` 除錯工具

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/dev/i2c-N` | i2c-dev 字元裝置，raw 存取 bus N（open + `I2C_SMBUS` ioctl） |
| `/sys/bus/i2c/devices/i2c-N/new_device` | `echo "driver addr"` 動態綁 driver |
| `/sys/bus/i2c/devices/i2c-N/delete_device` | 動態解綁 |
| `/dev/i2c-mux/...` | mux channel 選擇節點（部分平台暴露） |
| `/sys/class/hwmon/hwmonN/` | 感測器 driver 暴露的 value sysfs |

## 新手常問

- **怎麼看 bus 上有哪些 device？** `i2cdetect -y -r N`（`-r` 做 read）；不回應 standard 位址的 device 要自己讀，或先確認 mux channel 選對。
- **為什麼同一條物理線路對應多個 `/dev/i2c-N`？** mux 的子 channel 各拿新 adapter 編號；`i2cdetect -F N` 可顯示該 bus 上的 mux chip。
- **I3C 與 I2C 能混在同一條 bus？** 可以，I3C 向下相容 I2C：I3C master 能以 I2C 時鐘驅動 legacy I2C device；I2C device 保持固定位址，I3C device 用 ENTDAA 拿動態位址。
- **新增一顆感測器一定要改 device tree 嗎？** 不一定：(a) device tree 列 driver＋位址讓 kernel probe；(b) `new_device` 動態拉起；(c) entity-manager JSON 描述 Bus/Address 後由 daemon 做 raw SMBus 讀。OpenBMC 上 (c) 最常見。
- **`i2cget` 沒回應檢查什麼？** mux channel（預設常在 0 或 1）、GPIO gating（有些 bus 段要 GPIO 先切換）、以及 `i2cdetect` 看位址是否真有回應。
