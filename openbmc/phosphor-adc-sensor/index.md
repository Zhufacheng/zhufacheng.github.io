# phosphor-adc-sensor

`phosphor-adc-sensor` 讀 Linux IIO 匯出的 ADC channel，換算成實際電壓後，以標準 `Sensor.Value` 物件（通常 type 為 `voltage`）掛上 D-Bus。它負責把裸的 ADC raw 值變成 OpenBMC 通用感測器格式，讓其他 daemon 不用自己碰 IIO sysfs。

## 背景概念

- **IIO（Industrial I/O）**：Linux 的類比感測器框架，ADC 通道暴露到 `/sys/bus/iio/devices/`。
- **IIO ADC 的關鍵檔案**（`iio:deviceX` 下）：
  - `in_voltageY_raw` — 原始碼值
  - `in_voltageY_scale` — 換算比例
  - `in_voltageY_offset` — 偏移量
- **換算式**：`實際值 = raw × scale (+ offset)`，單位通常是伏特（或微伏，視驅動）。
- **感測器 D-Bus 物件**：`/xyz/openbmc_project/sensors/voltage/<Name>`，interface `xyz.openbmc_project.Sensor.Value`（`Value`、`Unit`、`MinValue`/`MaxValue`）。

## 怎麼運作

```
systemd 起 phosphor-adc-sensor.service
  │
  ├─ 建 D-Bus 連線、Object Manager 管 /xyz/openbmc_project/sensors 子樹
  ├─ 掃描 /sys/bus/iio/devices/
  │     對每個 IIO 裝置：
  │       ├─ 列 in_voltageY_raw / _scale / _offset
  │       └─ 依 ADC 通道建 voltage 感測器
  ├─ request_name(...) 並進 event loop
  │
  ▼ 之後（排程 poll，IIO 無 change 通知）
  定期讀 raw → raw×scale(+offset) → 寫 Sensor.Value
  新 IIO ADC 裝置出現 → 建對應感測器
```

1. **掃描 IIO**：列 `/sys/bus/iio/devices/`，找出每個 ADC 通道的 `in_voltageY_raw`、`in_voltageY_scale`、`in_voltageY_offset`。
2. **建感測器**：每顆 ADC 通道建一顆 `voltage` 感測器，掛 `/xyz/openbmc_project/sensors/voltage/<name>`，interface `Sensor.Value`。
3. **換算與發布**：定期讀 `raw`，套 `raw × scale (+ offset)` 換算成電壓，寫 `Value`。
4. **動態**：新 IIO ADC 裝置出現時建對應感測器。

> 因為 IIO 一般沒有「值改變」的中斷通知，ADC 感測器是**排程 poll**（跟純 D-Bus 事件驅動的 reading-monitor 不同），輪詢間隔由設定決定。

## 相關專案 / daemon

- `phosphor-adc-sensor` — 本 daemon
- 上游 IIO ADC 驅動 — 提供 `/sys/bus/iio/devices/` 的 channel
- `phosphor-reading-monitor` — 對這些電壓感測器做 threshold 告警
- `bmcweb` / `phosphor-ipmi-sensor` — 對上層呈現

## D-Bus

- **Object path**：`/xyz/openbmc_project/sensors/voltage/<Name>`（依通道命名），Object Manager 管理。
- **Interface**：`xyz.openbmc_project.Sensor.Value`
  - `Value` (double) — 換算後電壓
  - `Unit` (string) — 電壓單位
  - `MinValue` / `MaxValue` (double) — 有效範圍（有提供才設）
- **Well-known service**：daemon 註冊 `xyz.openbmc_project.*` well-known name 持有子樹；客戶端以 object path 發現感測器。
- 有 threshold 設定時可另掛 `xyz.openbmc_project.Sensor.Threshold.<Severity>`（判定由 `phosphor-reading-monitor` 負責）。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/sys/bus/iio/devices/iio:deviceX/` | IIO 裝置目錄（每裝置一個） |
| `.../in_voltageY_raw` | ADC 通道原始碼值 |
| `.../in_voltageY_scale` | 換算比例 |
| `.../in_voltageY_offset` | 偏移量 |
| `/xyz/openbmc_project/sensors/voltage/...`（D-Bus） | 本 daemon 發布的電壓感測器 |

## 新手常問

- **為什麼有的 ADC 讀到是 0 或亂值？** `scale` / `offset` 沒設定好、或該通道沒真正接上感測電路時，raw 值就無意義；換算出來的值要結合硬體定義判斷。
- **ADC 跟 hwmon 的 voltage 有什麼不同？** hwmon 的 `in*_input` 是驅動已經換算過的 mV 值；IIO ADC 常常是 raw + scale/offset，要 daemon 自己換算。來源與換算層級不同。
- **它是事件驅動嗎？** 不是。IIO 無 change 通知，ADC 感測器走排程 poll，更新頻率取決於 poll 間隔。
- **它會算 threshold 嗎？** 不會，只負責把 ADC 讀數變成標準 `Sensor.Value`；告警由 `phosphor-reading-monitor` 負責。
