# phosphor-psu-manager

`phosphor-psu-manager` 是 OpenBMC 的電源供應器（PSU）管理 daemon：從 entity-manager 讀取每顆 PSU 的硬體描述，經 I2C/PMBus 監控 PSU 的存在 / 狀態 / 冗餘，讀取每顆 PSU 的感測器（輸入 / 輸出電壓、電流、功率、溫度、fan tach），並全部以標準 D-Bus 物件發佈——Redfish 的 Chassis Power（PSU 清單、功耗）與感測器查詢都是讀它發佈的東西。

本文對應上游 `openbmc/phosphor-psu-manager` repo。

## 背景概念

- **PMBus**：電源模組（PSU、VRM）的標準化管理協定，SMBus 的超集，有標準化的 command 集（VIN / IIN / POUT / TEMP…）。PSU 的 bus、I2C address、要讀哪些感測器 command，都由 entity-manager 的 config 描述。
- **PSU label**：PSU 的 EEPROM 中一區固定格式內容，放製造商、型號、part number、序列號等資產資訊；daemon 讀出來後發佈成 inventory property。
- **Redundancy（冗餘）**：平台通常有多顆 PSU（1+1、N+1 或 load sharing）。「冗餘」指多出來的顆數能覆蓋一顆失效；每顆 PSU 自己是 active 還是 standby，讀自 PMBus 的 status 註冊 / 感測器。
- **Hot-swap**：PSU 失效時可以熱插拔更換；在 D-Bus / Redfish 上表現為 presence 狀態變化與 `HotSwappable` 類 property。

## 怎麼運作

```
systemd 起 phosphor-psu-manager
  │
  ├─ 監看 entity-manager 的 PSU config（PMBus 型 expose）
  │    每顆 PSU：bus / address / 名稱 / SensorList / threshold
  │
  ├─ 每顆 PSU：
  │    ├─ 讀 label（製造商 / part number / 序列號等）
  │    ├─ 建 inventory 物件（presence、I2cDevice、asset 資訊）
  │    ├─ 依 SensorList 建標準感測器物件、訂閱更新（event-driven）
  │    └─ 追蹤冗餘 / 狀態（active / standby，讀自 PMBus status）
  │
  ▼（之後全由 D-Bus 訊號驅動，沒有固定 poll 排程）
entity-manager config 變動 / 感測器值變動 / PSU 被拔掉
       │
       ▼
更新 D-Bus property、發 PropertiesChanged
       │
       ▼
bmcweb（Redfish Power / Sensors）與其他 reader
```

1. **設定來源**：PSU 是 entity-manager 硬體描述 JSON 裡的 `PMBus` 型 expose，帶 bus 編號、I2C address、名稱、`SensorList`（每個感測器：名稱、PMBus command / page、scale、threshold 等）。範例（實際欄位以平台 entity-manager JSON 為準）：

```json
{
  "Exposes": [
    {
      "Type": "PMBus",
      "Bus": 4,
      "Address": 60,
      "Name": "PSU0",
      "SensorList": [
        { "Name": "Input_Voltage" },
        { "Name": "Output_Power" }
      ]
    }
  ]
}
```

2. **Inventory**：daemon 為每顆 PSU 建一個 inventory 物件（`/xyz/openbmc_project/inventory/system/powersupplies/` 子樹下），含 presence、I2C 位置（bus / address）、label 讀出的資產資訊。PSU 被拔掉時，對應物件的 presence / 存在狀態跟著變。
3. **感測器**：`SensorList` 裡的每個感測器成為一顆標準感測器物件——路徑 `/xyz/openbmc_project/sensors/<Type>/<PSU名稱>_<感測器名稱>`（例如 `PSU0_Input_Voltage`、`PSU0_Output_Power`，實際命名以 config 為準），interface 是 `xyz.openbmc_project.Sensor.Value`。值更新是 event-driven：daemon 讀到 PMBus 值變化才寫 D-Bus。
4. **冗餘與故障**：每顆 PSU 的 active / standby 狀態隨 inventory 物件發佈（bmcweb 映射到 Redfish `PowerSupplies[].RedundancyMode`）。故障表達有兩層：
   - **PSU 層**：presence / 狀態 property（例如整顆 PSU 報 fault、掉出冗餘）
   - **感測器層**：單個量測的 threshold alarm（例如輸入電壓 critical low）
   
   bmcweb 把兩層合成 Redfish 的 `Status`（ok / warning / critical）。

## 相關專案 / daemon

- `entity-manager`：PSU config（bus / address / 感測器清單）的來源
- `phosphor-psu-manager`（本 repo）：daemon 本體
- `bmcweb`：Redfish Chassis Power（PSU 清單、`PowerConsumedWatts` 等）讀它發佈的物件
- `phosphor-sensor`：另一組感測器 reader（hwmon / eeprom 等）；PSU 的 PMBus 感測器歸 psu-manager 管，不重疊

## D-Bus

| 項目 | 值 |
|---|---|
| object（inventory） | `/xyz/openbmc_project/inventory/system/powersupplies/<PSU名稱>` |
| interface | `xyz.openbmc_project.Inventory.Item`（presence / 名稱）、`xyz.openbmc_project.Inventory.Decorator.I2cDevice`（Bus / Address）、asset 類 decorator（製造商 / 型號 / part number / 序列號） |
| object（感測器） | `/xyz/openbmc_project/sensors/<type>/<PSU名稱>_<感測器名稱>`（type：`voltage` / `current` / `power` / `temperature` / `fan_tach`） |
| interface | `xyz.openbmc_project.Sensor.Value`（Value / Unit / MinValue / MaxValue）＋ threshold 類 interface |
| 呼叫端 | 經 `/xyz/openbmc_project/ObjectMapper` 的 ObjectMapper 查物件歸屬哪個 service，不 hard-code well-known name |

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| entity-manager 的 PSU config JSON（`PMBus` 型 expose ＋ `SensorList`） | bus / address / 感測器清單的來源 |
| daemon 本身沒有本地 config 檔 | 一切設定來自 entity-manager（D-Bus），換硬體配置不用改 daemon |

## 新手常問

- **PSU 感測器怎麼不見了？** 通常是 entity-manager 沒把 PSU expose 出來（config 沒 match 到硬體），或 I2C 讀不到（bus / address 錯、PSU 未上電）。先查 entity-manager 的 D-Bus 輸出，再查 I2C bus。
- **PSU 層故障跟感測器 threshold 有什麼不同？** 感測器 threshold 是「單個量測值」越線（輸入電壓太低）；PSU 層狀態是 PSU 模組自己報的整體狀態（PMBus status）。Redfish `Status` 把兩者合成。
- **Redfish 怎麼知道這顆 PSU 現在吃多少瓦？** 讀 daemon 發佈的 power 感測器（input / output power），沒有另外一個「power 物件」。
- **PSU 被拔掉 daemon 會 crash 嗎？** 不會。掉線是狀態變化：inventory 的 presence 跟著變、感測器值依標準規則標記失效，不會中斷其他 PSU。
- **要新增一顆 PSU 的感測器怎麼做？** 改 entity-manager 的 PSU expose（`SensorList` 加一筆），重跑 entity-manager；daemon 端不用改。
