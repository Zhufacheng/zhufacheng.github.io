# phosphor-inventory

`phosphor-inventory` 是 OpenBMC 的硬體 inventory daemon：把系統上發現的硬體（主機板、CPU、記憶體、風扇、電源、drive…）以 D-Bus 物件發布，讓 consumer（`bmcweb`/Redfish、asset tracking、報障邏輯）用統一介面查「這台機器有哪些硬體、規格是什麼」。

## 背景概念

- **inventory 樹**：所有 inventory 物件都掛在 `/xyz/openbmc_project/inventory/` 下，路徑反映實體層級（system → chassis → motherboard → 元件）。
- **`Inventory.Item` 介面**：所有 inventory item 的 base interface；再疊加型別專屬介面 `Inventory.Item.<Type>`（`Board`、`Fan`、`PowerSupply`、`Drive`…），每個 type 有一組固定 properties。
- **ObjectMapper**（service `xyz.openbmc_project.ObjectMapper`）：consumer 不直接找 producer，而是問 ObjectMapper「`/xyz/openbmc_project/inventory/...` 子樹下哪些路徑、歸哪個 service」——producer 換掉，consumer 不用改。
- **producer 有多家**：`entity-manager`（現行主要）、`phosphor-inventory`（較早期的獨立 daemon）、`phosphor-software-manager`（firmware 版本）。

## 怎麼運作

```
硬體描述 JSON / FRU EEPROM / firmware 版本
        │
        ▼
producer daemon 發現硬體
（entity-manager 解析 JSON、phosphor-inventory 讀 FRU、
phosphor-software-manager 讀 firmware）
        │
        ▼
建 D-Bus 物件：Inventory.Item + Inventory.Item.<Type>（+ Association）
        │
        ▼
/xyz/openbmc_project/inventory/...（object manager 管理子樹）
        │
        ▼
consumer：bmcweb（Redfish Chassis/Systems/PCI…）、
asset/tracking 邏輯、其他 daemon（經 ObjectMapper 查詢）
```

### inventory 樹長什麼樣

```
/xyz/openbmc_project/inventory/
└── system/
    ├── chassis/
    │   └── motherboard/            Inventory.Item.Board
    │       ├── cpu0/
    │       ├── dimm0/
    │       ├── fan0/               Inventory.Item.Fan
    │       └── power-supply0/      Inventory.Item.PowerSupply
    ├── drive/                      Inventory.Item.Drive
    └── software/                   Inventory.Item.Software
        ├── bmc/                    xyz.openbmc_project.Software.BMC
        └── bios/                   xyz.openbmc_project.Software.BIOS
```

### `Inventory.Item.Board` 的常見 properties

| property | 意義 |
|---|---|
| `Manufacturer` | 製造商 |
| `Model` | 型號 |
| `SerialNumber` | 序號 |
| `PartNumber` | 料號 |
| `ExtraData` | JSON string；介面沒定義到的平台自訂欄位常放這裡（consumer 自行解析） |

base interface `Inventory.Item` 另有一兩個共用欄位（如 `PrettyName` / `PrettyType`，給 UI 顯示用）。其他 type（`Fan`、`PowerSupply`、`Drive`…）各有自己的 properties，以 `phosphor-dbus-interfaces` 裡的 `Inventory/Item/*.interface` 為準。

### 跟 entity-manager 的關係

- 現行 OpenBMC 的**主要 inventory producer 是 `entity-manager`**（service `xyz.openbmc_project.EntityManager`）：它解析機器專屬的硬體描述 JSON，直接發布 `/xyz/openbmc_project/inventory/system/...` 下的 `Inventory.Item.*` 物件；同一份 JSON 也產出 `xyz.openbmc_project.Configuration.<Type>` 的設定物件（sensor、fan、power 等 daemon 吃的是後者）。
- `phosphor-inventory` 是較早期的獨立 inventory daemon，特定平台/舊配置仍在使用；兩者發布到**同一棵樹、同一套介面**，consumer 端（ObjectMapper + interface）無差異。
- `phosphor-software-manager` 管「firmware 版本」這種非實體硬體的 inventory（`system/software/...`），跟硬體 inventory 互補。

## 相關專案 / daemon

- `phosphor-inventory`（本頁主角）
- `entity-manager`（現行主要 producer）
- `phosphor-software-manager`（firmware inventory）
- `phosphor-dbus-interfaces`（`Inventory/Item*.interface` 介面定義）
- `bmcweb`（把 inventory 映射成 Redfish）

## D-Bus

- 物件路徑：`/xyz/openbmc_project/inventory/...`（各 producer 各自 owns 自己建的子樹；well-known name 因 producer 而異）。
- 介面：`xyz.openbmc_project.Inventory.Item` + `xyz.openbmc_project.Inventory.Item.<Type>`（`Board` / `Fan` / `PowerSupply` / `Drive`…）。
- 常用搭配：`xyz.openbmc_project.Association.Definitions`（`Associations` property，描述 item 之間、item 與 sensor 之間的關聯）。
- 查詢：對 `xyz.openbmc_project.ObjectMapper` 下 `GetSubTree` / `GetObject`，不綁定特定 producer。

## 新手常問

- **誰決定 inventory 樹裡有哪些物件？** 平台的硬體描述（entity-manager JSON）或平台自訂的 producer 邏輯；同一台機器上多個 producer 的物件並存在同一棵樹下。
- **為什麼 consumer 不直接連 producer 的 well-known name？** producer 可能換（`phosphor-inventory` ↔ `entity-manager`）、物件可能分散在多個 daemon；統一走 ObjectMapper 解析「路徑 → service」。
- **`ExtraData` 是什麼？** 一個 JSON string，放介面沒定義到的平台自訂欄位（例如 FRU 裡的 extra bytes）。
- **firmware 版本也算 inventory 嗎？** 算，但 producer 是 `phosphor-software-manager`（`Inventory.Item.Software` + `Software.BMC`/`Software.BIOS`），不屬本頁範疇。
- **inventory 是即時量出來的嗎？** 不是——大多在開機時從硬體描述/FRU 建好，之後很少變動；它描述「有哪些硬體」，不是「硬體現在的狀態」（狀態走 sensor / alarm）。
