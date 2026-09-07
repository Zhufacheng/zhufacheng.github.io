# phosphor-virtual-sensor

phosphor-virtual-sensor 是 OpenBMC 社群（上游）的「虛擬感測器」daemon：把**已經存在的感測器**（或常數）拿出來做運算（乘除偏移、取中位數、取最大值...），算出來的結果當成一顆新的感測器，用標準感測器格式掛上 D-Bus。

它自己**不讀任何硬體**——輸入全部來自 D-Bus 上其他 daemon 已經公佈的感測器。典型用途：

- 進氣溫度 = 出氣溫度 − 溫差偏移
- 某組溫度只信任其中位數（ModifiedMedian，壞一顆不誤報）
- 風扇 tach 換算成其他單位

這份文件對應上游 `openbmc/phosphor-virtual-sensor` repo（onetree 的 recipe `phosphor-virtual-sensor_git.bb` 從 GitHub 抓的就是它）。

## 背景概念（先認識幾個名詞）

- **感測器 D-Bus 物件**：OpenBMC 的感測器一律掛在 `/xyz/openbmc_project/sensors/<Type>/<Name>`（Type 例如 `temperature`、`fan_tach`、`voltage`），interface 是 `xyz.openbmc_project.Sensor.Value`，主要 property 是 `Value`（數值）、`Unit`、`MinValue`/`MaxValue`。**虛擬感測器的輸出也是這種標準物件**——對使用者來得跟真的感測器沒兩樣。
- **虛擬感測器（virtual sensor）**：值不是量出來的、是**算出來的**感測器。輸入 = 其他感測器 + 常數，運算 = 一個數學表達式或限定好的演算法。
- **exprtk**：C++ header-only 的數學表達式引擎。虛擬感測器可以寫任意 exprtk 表達式（`P1 * (P2 + 5 - P3 * 0.01)` 這種），開機時 compile 一次，之後每次更新只要把變數值填進去、求值。
- **Threshold（threshold）與 Hysteresis（hysteresis）**：感測器可以掛 threshold interface（Warning、Critical、SoftShutdown、HardShutdown、PerformanceLoss 五種嚴重度），各有 high/low 兩條線。值越過線就 assert alarm、掉回線內**要退回 hysteresis 距離內**才 deassert——避免值在臨界點附近抖動時 alarm 一直亮滅。
- **兩種設定來源**：
  1. **靜態 JSON 檔**（`virtual_sensor_config.json`）：內容固定、可以寫任意 exprtk 表達式。
  2. **entity-manager（D-Bus）**：設定放在 entity-manager 的硬體描述 JSON 裡、走 D-Bus 過來；換算方式限定（目前只有 `ModifiedMedian` / `Maximum` 兩種），好處是不同硬體配置可以有不同虛擬感測器。

## 整體流程

```
systemd 起 phosphor-virtual-sensor.service (Type=dbus)
  │
  ├─ main()
  │    ├─ object manager 管 /xyz/openbmc_project/sensors 子樹
  │    ├─ VirtualSensors(bus)          ← 建構子就載入設定、建感測器
  │    │    └─ createVirtualSensors()
  │    │         ├─ 靜態 JSON 檔的 entry：直接建 VirtualSensor
  │    │         └─ Desc.Config="D-Bus" 的 entry：
  │    │              ├─ setupMatches()  盯 entity-manager 的
  │    │              │                  ModifiedMedian/Maximum interface
  │    │              └─ createVirtualSensorsFromDBus()
  │    │                   └─ GetManagedObjects 問 entity-manager
  │    ├─ request_name("xyz.openbmc_project.VirtualSensor")
  │    └─ process_loop()
  │
  ▼ （之後全由 D-Bus 訊號驅動，沒有排程 poll）
  來源感測器 Value 改變 / 消失 / 提供它的 service 掉線
         │
         ▼
  DbusSensor 訊號 handler → value 更新（或標 NaN）
         │
         ▼
  VirtualSensor::updateVirtualSensor()
     ├─ 把所有參數值填進 exprtk 變數
     ├─ 求值（expression / ModifiedMedian / Maximum）
     ├─ clamp 到 MinValue/MaxValue → 寫 Sensor.Value
     └─ checkThresholds() 五種 threshold 依序查
          （越線 assert + log；退回 hysteresis 內 deassert）
```

## 主體怎麼運作（依序）

### 1. 開機（`main()`）

1. 建 D-Bus 連線、object manager 管 `/xyz/openbmc_project/sensors` 子樹。
2. `VirtualSensors(bus)`：**建構子就載入設定並建立所有虛擬感測器**（所以 daemon 起來後感測器馬上就在 D-Bus 上）。
3. `request_name("xyz.openbmc_project.VirtualSensor")`；`process_loop()` 進 event loop。
4. systemd unit 是 `Type=dbus`、`Restart=always`。

### 2. 載入設定：`VirtualSensors::createVirtualSensors()`

對設定裡的每筆 entry 分兩路：

**路徑 A：靜態 JSON 檔**

`parseConfigFile()` 依序找 `virtual_sensor_config.json`：

1. 目前工作目錄
2. `/var/lib/phosphor-virtual-sensor/`
3. `/usr/share/phosphor-virtual-sensor/`（build 時裝的 sample）

每筆 entry 的欄位（`virtual_sensor_config.json` 的範例改寫）：

```json
{
    "Desc": {
        "Name": "Virtual_Inlet_Temp",     // 感測器名（空格會被換成 _）
        "SensorType": "temperature",       // 決定掛哪種 unit
        "MaxValue": 127.0, "MinValue": -128.0
    },
    "Threshold": {
        "CriticalHigh": 90, "CriticalLow": 20,
        "WarningHigh": 70,  "WarningLow": 30
    },
    "Associations": [["chassis", "all_sensors", "/xyz/.../my_board"]],
    "Params": {
        "ConstParam": [ { "ParamName": "P1", "Value": 1.1 } ],
        "DbusParam":  [
            { "ParamName": "P2",
              "Desc": { "Name": "MB_INLET_TEMP", "SensorType": "temperature" } }
        ]
    },
    "Expression": "P1 * (P2 + 5)"
}
```

- `Desc.SensorType` 要落在支援清單：`temperature, fan_tach, fan_pwm, voltage, altitude, current, power, energy, utilization, airflow, pressure`（對應各別的 Unit）。
- `Params.ConstParam`：常數參數。
- `Params.DbusParam`：指向另一顆 D-Bus 感測器（路徑 = `/xyz/openbmc_project/sensors/<SensorType>/<Name>`）。
- `Expression`：exprtk 表達式（string，或 string 陣列——會接起來）。
- `Threshold` / `Associations`：可選（見第 6、7 節）。

建好後：`updateVirtualSensor()` 先算一次、設 `Unit`、`emit_object_added()` 正式上線。同名感測器直接跳過（log 錯誤）。

**路徑 B：entity-manager（D-Bus）**

entry 長這樣：

```json
{ "Desc": { "Config": "D-Bus", "Type": "ModifiedMedian" } }
```

表示「到 entity-manager 找 `xyz.openbmc_project.Configuration.ModifiedMedian` 型的設定」。動作：

1. **`setupMatches()`**：先掛 `PropertiesChanged` match 盯 `/xyz/openbmc_project/inventory` 下的兩個計算 interface（`Configuration.ModifiedMedian`、`Configuration.Maximum`）——**因為 entity-manager 可能還沒起來，要先掛好才不會漏掉**。
2. `createVirtualSensorsFromDBus()`：向 entity-manager 做 `GetManagedObjects`（它還沒起來就當空處理，之後靠 match 補）。對每個有計算 interface 的物件：
   - 名字 = 路徑最後一段；`Units` property 換回 sensorType；
   - 建 `VirtualSensor`（D-Bus 版）：`Sensors` 欄位列的每顆感測器變參數、`Thresholds*` 子 interface 變 threshold、`MaxValidInput`/`MinValidInput` 決定哪些輸入值算有效；
   - 自動加 association：`chassis → all_sensors → <entity 的 parent path>`；
   - 再掛該 entity path 的 `InterfacesRemoved` match——**entity-manager 把這筆設定移除時，虛擬感測器跟著從 map 刪掉**。
3. `propertiesChanged` 回呼收到多個 callback（一顆感測器的多個 interface 各自發），所以只用「property 含 `Type`」的那次當有效觸發。

> D-Bus 格式的欄位規定在 entity-manager 的 `schemas/virtual_sensor.json`。

### 3. 輸入參數：`SensorParam` 與 `DbusSensor`

每個參數（常數或 D-Bus 感測器）是一個 `SensorParam`，名字同時是 exprtk 的變數名。D-Bus 型的包一個 `DbusSensor`，它是**事件驅動**的（不 poll）：

- 建的時候：向 ObjectMapper 查這顆感測器歸哪個 service、`Get` 一次 `Value`（查不到就 NaN）；訂閱該 service 的 `NameOwnerChanged`。
- **`Value` property 改變** → 存下新值（非 finite 值當 NaN）→ 立刻 `updateVirtualSensor()`。
- **InterfacesRemoved**（這顆感測器被移除）→ 值設 NaN → `updateVirtualSensor()`。
- **service 掉線**（NameOwnerChanged 舊 owner 沒了）→ 值設 NaN → `updateVirtualSensor()`。

所以「來源感測器變了 → 虛擬感測器重算」是同步的、不用等排程。

### 4. 算值：`VirtualSensor::updateVirtualSensor()`

1. 把每個參數的現值填進 exprtk 變數（`getParamValue()`：常數直接回、D-Bus 型回 cache 的值）。
2. 求值：
   - 一般（JSON 檔）設定：`expression.value()`——直接算 exprtk 表達式。
   - `ModifiedMedian`（D-Bus 設定）：`calculateModifiedMedianValue()`——取**有效範圍內**（`MinValidInput`~`MaxValidInput`）的輸入值排序：偶數個取中間兩個的平均、奇數個取中間那個、**只剩 2 顆取較大值**、0 顆回 NaN。（「modified」的重點：壞掉的感測器會超出有效範圍被剔除，剩下的仍給出合理中位數。）
   - `Maximum`：`calculateMaximumValue()`——有效範圍內的最大值，一顆都沒有就 NaN。
3. `setSensorValue()`：clamp 到 `MinValue`/`MaxValue` 後寫 `Sensor.Value`。
4. 跑第 5 節的 threshold 檢查。

exprtk 變數表除了參數，還註冊了常數、向量運算 package、和三個**耐 NaN 的自訂函式**（可以在表達式裡用）：

| 函式 | 作用 |
|---|---|
| `maxIgnoreNaN(a, b, ...)` | 最大值，忽略 NaN 的參數 |
| `sumIgnoreNaN(a, b, ...)` | 加總，忽略 NaN 的參數 |
| `ifNan(a, b)` | a 是 NaN 就回 b，否則回 a |

### 5. 發布到 D-Bus

每顆虛擬感測器 = 一個 D-Bus 物件，路徑 `/xyz/openbmc_project/sensors/<Type>/<Name>`，掛：

- `Sensor.Value`：`Value`、`Unit`、`MinValue`/`MaxValue`（設定有給才設）。
- `Sensor.Threshold.*`：只有設定裡**有該值**的嚴重度才建（見下節）。
- `Association.Definitions`：JSON 檔有 `Associations` 就照列的建；D-Bus 設定自動建 `chassis → all_sensors → <entity parent path>`。

### 6. Threshold：`createThresholds()` + `checkThresholds()`

- `createThresholds()`：對 Critical / Warning / HardShutdown / SoftShutdown / PerformanceLoss 五種，只要 config 有該 High 或 Low 值就建對應 interface，帶 `High`/`Low` + `HighHysteresis`/`LowHysteresis`（沒給就是 0）。
- `checkThresholds()`（每次 update 都跑五種）：
  - `value >= High`（或 `<= Low`）且 alarm 沒亮 → **assert**：`alarmHigh=true`、發 `HighAlarmAsserted` 訊號、log error。
  - alarm 亮著、`value < High - Hysteresis`（或 `> Low + Hysteresis`）→ **deassert**、發 `...Deasserted`、log info。
- 額外行為（Warning / Critical）：若這顆虛擬感測器來自 D-Bus 設定、且 config 有記對應的 entity-manager interface（`<Severity><High/Low>Direction`），**setter 被呼叫時會把新值寫回 entity-manager 的那個 interface**（`setDbusProperty` 到 EntityManager）——runtime 改 threshold 能持久化回硬體描述。

### 7. 重要檔案

| 路徑 | 用途 |
|---|---|
| `virtual_sensor_config.json`（cwd / `/var/lib/phosphor-virtual-sensor/` / `/usr/share/phosphor-virtual-sensor/`） | 靜態設定；依序找，找到就用 |
| `/usr/share/phosphor-virtual-sensor/virtual_sensor_config.json` | build 裝的 sample |
| entity-manager 的硬體描述 JSON（`Configuration.ModifiedMedian` / `Configuration.Maximum` 型 expose） | D-Bus 設定來源 |
| `schemas/virtual_sensor.json`（entity-manager repo） | D-Bus 設定的格式規定 |

## 新手常問

- **跟 phosphor-sensor 有什麼差別？** phosphor-sensor 負責「量」（讀 hwmon/eeprom 等真實來源）；virtual-sensor 負責「算」（把量出來的東西組合出新感測器）。兩者輸出都是同格式的 `Sensor.Value` 物件。
- **為什麼有的感測器值變成 NaN？** 來源感測器被移除、提供它的 service 掉線、或（ModifiedMedian/Maximum）所有輸入都超出 `MinValidInput`~`MaxValidInput` 有效範圍——都回 NaN，表示「這個值現在不可信」。
- **ModifiedMedian 壞兩顆以上會怎樣？** 超出有效範圍的輸入被剔除，剩下的照常算中位數；全部剔除（或一顆都不剩）就 NaN。
- **表達式寫錯會怎樣？** 開機 compile 失敗 → log 每個 token 的錯誤位置、丟 exception，這顆感測器建不出來（其他的不受影響）。
- **為什麼 D-Bus 設定只能用 ModifiedMedian/Maximum？** 上游刻意把 D-Bus 動態設定的運算限定在兩種受限演算法（避免任意表達式進 entity-manager 的設定）；要任意 exprtk 表達式請走靜態 JSON 檔。
- **要新增一顆虛擬感測器怎麼做？** 寫一筆 JSON（`Desc.Name/SensorType`、`Params`、`Expression`）放進配置檔重新部署；或要跟著硬體配置走的話，放進 entity-manager 的 JSON 用 `ModifiedMedian`/`Maximum` 型。
