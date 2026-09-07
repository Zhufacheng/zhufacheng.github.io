# FRU (Field Replaceable Unit)

FRU 是 IPMI 定義的一套**小型 EEPROM 資料格式**，記錄「可現場更換的單元」（主機板、電源、風扇模組、背板…）的製造商、序號、料號等資訊。OpenBMC 的 BMC 從 I2C 讀這些 EEPROM、把解析結果發上 D-Bus，供 IPMI FRU 命令、Redfish 的 asset 欄位和 inventory 使用。

## 背景概念

- **FRU EEPROM**：一顆小 I2C EEPROM（常見 256B / 512B / 1KB / 2KB），位置 = (I2C bus, 7-bit address)，由平台硬體描述 / FRU locator 定義。
- **FRU Device ID**：IPMI 層面用 `FRU Device ID` 選目標（0 = baseboard）；`ipmitool fru print 0` 讀的就是 ID 0。
- **欄位格式**：每個欄位 = `field code (1B) + type (1B) + length (1B) + value (len B)`，type 幾乎都是 `0x08`（ASCII）。

## 怎麼運作

### EEPROM 內的 layout

```
offset 0
┌─────────────────────────────┐
│ FRU header (16 bytes)       │  format ID(=0)、version、checksum、
│                             │  custom(manufacturer) area pointer、
│                             │  board / chassis / product area offsets
├─────────────────────────────┤
│ board area                  │  ← 三個 area offset 以 board area 起點
│   [3B area header]          │    為基準（相對 offset）
│   field code/type/len/value │    一連串欄位（mfg date、mfg、name、
│   ...                       │    serial、part number…）
├─────────────────────────────┤
│ chassis area                │  同上結構
├─────────────────────────────┤
│ product area                │  同上結構
├─────────────────────────────┤
│ manufacturer (custom) area  │  位置由 header 的 custom pointer 決定
└─────────────────────────────┘
```

- 每個 area 的 3-byte header：length low / length high / checksum（該 area 所有 byte 的 two's-complement 和）。
- 常用 field code（完整表見 IPMI 2.0 spec 的 FRU 定義）：

| area | code | 欄位 |
|---|---|---|
| board | 0x00 | Manufacturing Date |
| board | 0x01 | Manufacturer |
| board | 0x02 | Product Name |
| board | 0x03 | Serial Number |
| board | 0x04 | Part Number |
| board | 0x07 | Language Code |
| product | 0x00 | Manufacturer |
| product | 0x01 | Product Name |
| product | 0x02 | Part Number |
| product | 0x03 | Version |
| product | 0x04 | Serial Number |
| product | 0x05 | Asset Tag |
| chassis | 0x00 | Part Number |
| chassis | 0x01 | Serial Number |

### 讀寫路徑

```
FRU EEPROM (I2C bus N, addr 0x20)
        │  I2C read
        ▼
phosphor-fru-device 解析 layout（header → 各 area → 欄位）
        │  D-Bus 發布（每顆 FRU 一個物件，含原始 byte）
        ▼
┌──────────────────┬───────────────────┬──────────────────────┐
│ phosphor-ipmi-host│ inventory         │ bmcweb (Redfish)     │
│ (IPMI FRU 命令)   │ (Inventory.Item)  │ (asset 欄位)         │
└──────────────────┴───────────────────┴──────────────────────┘
```

- `phosphor-fru-device` 開機後從平台的 FRU 位置資訊（`(bus, address)`，通常寫在 entity-manager 的硬體描述 JSON 裡）拿到每顆 FRU EEPROM 的位置，I2C 讀出整顆、解析後把 FRU 內容（含原始 EEPROM byte）以 D-Bus 物件發布（`xyz.openbmc_project.FRUDevice` service）。
- **IPMI 路徑**：`phosphor-ipmi-host` 實作 FRU netfn 的命令（`Get FRU Data` / `Set FRU Data` / `Get FRU Inventory Area Information` / `Get FRU Inventory Device Information`），讀寫來源是 D-Bus 上的 FRU 物件，不是直接碰 I2C——所以 `ipmitool fru read/writemem` 走的也是同一份資料。
- **inventory 路徑**：FRU 欄位餵進 inventory item（例如 `Inventory.Item.Board` 的 `Manufacturer` / `Model` / `SerialNumber` / `PartNumber`，或放 `ExtraData`）；具體怎麼映射由平台的 producer（entity-manager JSON 或自訂邏輯）決定。
- **Redfish 路徑**：Redfish 沒有標準 FRU resource；FRU 資料以 asset 欄位形式進 Redfish（`ComputerSystem` / `Chassis` / `Manager` 的 `Manufacturer`、`SerialNumber`、`PartNumber` 等），上游常是 FRU → inventory → `bmcweb`。
- **寫**：`Set FRU Data`（`ipmitool fru writemem`）→ `phosphor-ipmi-host` → D-Bus → `phosphor-fru-device` → I2C 寫回 EEPROM。

## 相關專案 / daemon

- `phosphor-fru-device`：讀/寫 FRU EEPROM、解析 layout、發布 D-Bus 物件
- `phosphor-ipmi-host`：IPMI FRU 命令 handler
- `bmcweb`：Redfish asset 欄位（序號/料號的上游常是 FRU）
- `entity-manager`：提供 FRU EEPROM 的 (bus, address) 位置資訊（硬體描述 JSON）

## D-Bus

- service `xyz.openbmc_project.FRUDevice`：每顆 FRU 一個 D-Bus 物件，含解析後的欄位與原始 EEPROM byte；IPMI 與 Redfish 的 FRU 讀取都以它為單一資料源。
- 下游消費：inventory（`Inventory.Item.*`）讀的是同一份解析結果。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| FRU EEPROM（I2C bus + 7-bit address，如 bus 2 / `0x20`） | 實體資料來源，layout 如上文 |
| entity-manager 硬體描述 JSON（平台 repo 安裝） | 每顆 FRU EEPROM 的 (bus, address) 位置 |
| IPMI FRU Device ID（邏輯 id，0 = baseboard） | `ipmitool fru ...` 選擇目標 FRU |

## 新手常問

- **FRU 和 inventory 是什麼關係？** FRU 是「資料格式 + 那顆 EEPROM」；inventory 是 D-Bus 上的物件模型。FRU 欄位是 inventory 欄位（序號、料號…）的常見來源之一，但不是唯一來源（平台也可以直接寫進硬體描述）。
- **baseboard 的 FRU Device ID 一定是 0 嗎？** IPMI 慣例 0 = baseboard FRU；其他 ID 由平台配置決定。
- **改 FRU 序號要重開機嗎？** 不用，`Set FRU Data` 直接寫 EEPROM，D-Bus 上的內容跟著更新。
- **EEPROM 讀不到會怎樣？** 該 FRU 不會有（或只有空）D-Bus 物件，對應 inventory 欄位缺值，IPMI 命令回 read error。
- **FRU 和 SMBIOS 是什麼關係？** 不同格式、不同位置：FRU 在 BMC 側 EEPROM、BMC 自己讀；SMBIOS 在 host 側（BIOS 提供），BMC 不負責維護。
