# entity-manager

entity-manager 是 OpenBMC（伺服器 BMC 韌體）裡的「硬體清點」元件：開機時（以及之後硬體有變化時），它判斷「這台機器上實際有哪些硬體」，把結果用 D-Bus 公佈出來，給系統裡的其他軟體使用（風扇控制、感測器、IPMI、SDR、Web UI...）。

這份文件講的是 onetree 樹 vendor layer `recipes-phosphor/configuration/entity-manager/` 下的版本（完整 source 在同目錄的 fork 目錄裡）。一個 recipe 建出三個 daemon：

| daemon | 工作 |
|---|---|
| `entity-manager` | 主體：讀 JSON 硬體描述 → 對照 D-Bus 上實際存在的東西 → 公佈 inventory（硬體清單） |
| `fru-device` | 掃 i2c 上的 FRU EEPROM（產品資訊），解析後公佈到 D-Bus |
| `devicetree-vpd-parser` | 從 device tree 讀機器資訊（machine context）公佈到 D-Bus |

> 文中 `<T>` 代表 /tmp 下的一個 vendor 子目錄（確切名稱見 code）。

## 背景概念（先認識幾個名詞）

- **D-Bus**：Linux 系統上的「訊息總線」。各服務把資料掛成「物件」，有路徑（長得像 `/xyz/openbmc_project/inventory/system/board/...`）、有 interface、有 property。任何程式都可以讀這些 property（少數可以寫）。**entity-manager 的輸出全是 D-Bus 物件**——它自己不直接控制任何硬體，只負責「公佈事實」。
- **配置 JSON**：對「這型機器*可能*有哪些硬體」的描述檔，放在 `/usr/share/entity-manager/configurations/`。每檔一筆或多筆，長這樣（示意）：
  ```json
  {
      "Name": "PDB Temp",
      "Type": "TMP75",
      "Probe": "xyz.openbmc_project.Configuration.Adc('Name' = \"PDB Temp\")",
      "Bus": 11,
      "Address": "0x49",
      "Exposes": [ ... 如果它在，要公佈哪些子項目 ... ]
  }
  ```
  关键字段：
  - `Name` / `Type`：這筆硬體的名字與類型。
  - `Probe`：判斷「它到底有沒有在」的條件（可有多個，全部符合才算在）。
  - `Exposes`：陣列，列出它底下要公佈的子項目（感測器、Mux、PSU...）。
- **Probe**：一個可以對「D-Bus 上實際的東西」做驗證的條件。最常見的形式是「某個 interface 的某個 property 符合某值」，例如 `xyz.openbmc_project.Configuration.CpuStatus('Present' = true)`——「只要 D-Bus 上有 CpuStatus 且 Present=true，這筆就算偵測到」。
- **Inventory**：最終產物——「這台機器*現在*有哪些硬體」的完整清單，全部掛在 D-Bus 的 `/xyz/openbmc_project/inventory` 子樹下。
- **FRU**：標準格式（IPMI FRU）的產品資訊，存在小 EEPROM 裡：製造商、型號、序號、asset tag 等。`fru-device` 專門讀這個。

## 主體（entity-manager）怎麼運作

### 1. 開機

`main()` 做幾件事：

1. 抓 D-Bus bus name `xyz.openbmc_project.EntityManager`；object manager 只管 `/xyz/openbmc_project/inventory` 子樹（其他路徑的物件不归它管）。
2. 掛一個 `ReScan` method——外部呼叫它就重跑一遍完整掃描。
3. 註冊三種 D-Bus signal match，作為「該重掃了」的觸發：
   - `NameOwnerChanged`：任何服務上下線（新 daemon 起來可能帶來新硬體資訊）。
   - `InterfacesAdded` / `InterfacesRemoved`：只看 payload 有沒有跟「所有 probe 用到的 interface」相交（開機先用 `getProbeInterfaces()` 把這些 interface 名收集好）。
4. `setupPowerMatch()`：盯 host 電源狀態（`/xyz/openbmc_project/state/host0` 的 `CurrentHostState`），記住 host 有沒有開機——這會影響「device 消失」該怎麼處理（見第 8 節）。
5. `fwVersionIsSame()`：把 `/etc/os-release` 全文做 hash，跟 `/var/configuration/version` 存的比。韌體沒換過（或上次有存過 config），就把上次的 `/var/configuration/system.json` 搬到 `/tmp/configuration/last.json` 讀進 `lastJson`——這是「上次開機狀態」，用來保留 runtime 改過的值（asset tag、PID 參數等，見第 7 節）。
6. `io.run()`：之後全靠 signal 驅動。

### 2. 重掃的入口：`propertiesChangedCallback()`

任何觸發都走同一個入口：

- **500ms debounce**：用 timer 把密集訊號合成一輪（timer 被新的請求 cancel 就重排）。
- **重入保護**：上一輪還沒跑完，這輪就再排一次，不併行跑。
- 一輪的動作：`loadConfigurations()`（重讀所有 JSON）→ 起 `PerformScan` → scan 全部結束後做「移除 + 發布」。

另外 `getInterfaces()` 抓到每個 probe path 時，會對該 path 註冊 `PropertiesChanged` match——**被 probe 的 device 自己改 property 也會觸發重掃**（例如 CPU 狀態改變）。

### 3. 讀配置：`loadConfigurations()`

讀兩個目錄的所有 `*.json`：

- `/usr/share/entity-manager/configurations`（vendor 各平台 JSON 裝在這）
- `/etc/entity-manager/configurations`（host 端可覆寫）

JSON 是 array 就拆成多筆。schema 驗證（valijson + `global.json`）在 code 裡目前註解掉，不驗。

### 4. 掃描：找出 D-Bus 上相關的東西（PerformScan）

掃描分三步，全部是問 D-Bus「現在有什麼」：

1. **`PerformScan::run()`**：把每筆 config 的 `Probe` 建一個 `PerformProbe` 物件；同時從 probe 字串裡抽出「要查哪些 D-Bus interface」（`(` 前面的部分）。
2. **`findDbusObjects()`**：向 `xyz.openbmc_project.ObjectMapper` 做 `GetSubTree`，一次問回「系統上哪些 path 暴露這些 interface」。失敗重試 5 次（間隔 10s）。
3. **`getInterfaces()`**：對每個 (path, interface) 做 `GetAll`，把該 interface 的所有 property 值抓回來存著（`dbusProbeObjects[path][interface]`），後面 probe 比對和 template 取代都用這份快照。失敗 2s 重試、最多 5 次。

### 5. Probe：判斷「這筆硬體在不在」（PerformProbe）

probe 是一個「小程式」，由 `doProbe()` 依序評估 probe 陣列裡的每一句：

| 句子 | 效果 |
|---|---|
| `TRUE` / `FALSE` | 直接給定真/假 |
| `AND` / `OR` | 跟前面結果合併（`前 AND 本` / `前 OR 本`） |
| `FOUND('某probe名')` | 看之前有沒有 probe 通過過（讓後面的 config 依賴前面的偵測結果） |
| `MATCH_ONE` | 通過時只留「最後一個」match 的 device |
| `interface名('Prop' = 值, ...)` | D-Bus probe：去比對抓回來的 property |

D-Bus probe 的比對規則（`probeDbus()` + `matchProbe()`）：

- string 對 string：走 **regex**（所以可以寫 `('Name' = "CPU[0-9]")` 這種）。
- 數字/布林等：直接**相等**。
- 列出的每個 property 都要符合才算這筆 device 存在；符合的 (path + properties) 記下來。

每筆 config 的 probe 是「一個通過一個」的鏈式結構：`PerformProbe` 被析構時若通過就更新系統配置，`PerformScan` 被析構時若這輪有通過就複製一份自己（帶上已通過的 probe 名單）再跑下一輪——這樣 `FOUND` 才能引用先前的結果。全部跑完才進下一步。

### 6. 合併進「系統配置」：`updateSystemConfiguration()`

通過 probe 的 config 要變成「這台機器上真實存在的一筆 device」，做幾件事：

- **`templateCharReplace()`**：JSON 裡的 template 變數換成真值。`$BUS`、`$ADDRESS`、`$index` 會換成該 device 在 D-Bus 上的 property 值；還支援簡單算式，例如 `$ADDRESS+1`。整串就是一個變數時直接換成原始型別（不用字串）。
- **`generateDeviceName()`**：`Name` 也當 template 處理；若換完跟已存在的名字重名，改用「第幾個發現的」數字再換一次去重。
- **`getRecordName()`**：probe 名 + device 內容 hash，當這筆 device 在記憶體裡的 key。
- **`applyExposeActions()`**：expose 裡可以寫 `BindXXX: "Name"`（把別筆 config 的 expose 抓過來嵌進這筆、Status=okay）或 `DisableNode: "Name"`（把目标 expose 設成 disabled）。
- **舊 device 直接保留**：如果這筆 device 上次就在（現有配置或 `lastJson` 裡有），原樣搬過來——**runtime 用 D-Bus 改過的值不會因為重新 probe 被蓋掉**。
- 記 `FoundProbePath`：這筆 device 是從哪個 D-Bus path 發現的（asset tag 寫入會用到）。

### 7. 發布到 D-Bus：`postToDbus()`

這是「清點結果」正式上線的地方。對每個 board：

- **路徑**：`/xyz/openbmc_project/inventory/system/<boardType 小寫>/<Name>`（非法字元換 `_`）。
- **同名 board 直接忽略**（vendor 的 crash fix——以前會 crash）。
- **掛的 interface**：
  - board 層：`Inventory.Item` + `Inventory.Item.<Type>`（如 `Inventory.Item.Board`）；
  - 每個 `Exposes` item（`Status: "disabled"` 跳過）：路徑再接 `/<ItemName>`，掛 `Configuration.<Type>`；item 內的 object 再各掛 `Configuration.<Type>.<Prop>`，array 則 `...<Prop>0/1/2...`；
  - `BMC` / `System` 型 item 另加對應的 Inventory interface。
- **哪些 property 可以寫**：預設全部 read-only；interface 名落在這七種的例外，是 **readWrite**：
  `FanProfile, Pid, Pid.Zone, Stepwise, Thresholds, Polling, Preserve`
  這是「不重刷韌體就熱改參數」的通道（例如 PID 參數），asset tag 的 `Inventory.Decorator.AssetTag` 也是 readWrite。寫入的值會同步進記憶體配置和快照檔，**重開機保留**。
- **`AddObject` / `Delete`**：每個 board 可 runtime 加新 expose（要通過 `schemas/<type>.json` 驗證）；readWrite 的 interface 可 `Delete`（該 config 設 null、移除 interface、寫快照）。
- **關聯（association）**：`Topology` 記錄 board 間的埠關係（`DownstreamPort` 的 `ConnectsToType` 對 upstream port、`PowerPort` 標記供電），發布成每個 board 的 `Association.Definitions`：`contained_by`/`containing`（誰在誰裡面）、`powered_by`/`powering`（誰供誰電）。
- **取值一律用記憶體裡的 `systemConfiguration`**（不是原始 JSON）——同樣是為了保留 runtime 改過的值。

### 8. 移除消失的 device

- **`pruneConfiguration()`**：這輪沒探到、又不是「上次就有」的 device，移除它所有 D-Bus interface、清 board 記錄。
- **host 關機時的特例**：config 標了 `PowerState: "On"` 的 device 只有 host 開機才看得到，所以 host 沒開機時**不 prune**（不知道是沒了還是只是沒開機）。
- **`startRemovedTimer()`**：「上次有、這次沒」的 device 不馬上刪，等 **10 秒**才移除——避免 power cycle 期間短暫讀不到就亂刪。

### 9. 持久化與 overlay

- **`writeJsonFiles()`**：每輪發布都把整個 `systemConfiguration` dump 到 `/var/configuration/system.json`——下次開機 lastJson 的來源，runtime 改動因此跨開機。
- **`loadOverlays()`**：JSON 除了「公佈事實」，還負責「把 i2c device 在 kernel 裡建出來」。expose 的 `Type` 若在 `devices::exportTemplates` 白名單（EEPROM、TMP75、Mux 等），就寫 `/sys/bus/i2c/devices/i2c-<Bus>/new_device` 實例化 driver；要求有 `hwmon` 子目錄的 device 要驗到才算成功，失敗就 `delete_device` 重試（500ms × 5 次）。
- **`linkMux()`**：Mux 型 device 建好後，把 `/sys/.../channel-N` 指向的 bus 建 symlink 到 `/dev/i2c-mux/<mux名>/<channel名>`——之後設定檔就能用 channel 名當 bus 用。

### 主體流程圖（對照上面各節）

```
systemd 起 (Type=dbus, BusName=xyz.openbmc_project.EntityManager)
  │
  ├─ main()               抓 bus name、object manager 管
  │                       /xyz/openbmc_project/inventory 子樹
  ├─ 註冊 3 種 D-Bus match：NameOwnerChanged、
  │   InterfacesAdded / InterfacesRemoved（只看 probe 用到的 interface）
  ├─ setupPowerMatch()    盯 host power 狀態
  ├─ fwVersionIsSame()    /etc/os-release hash 沒變 →
  │                       把 /var/configuration/system.json 搬到
  │                       /tmp/configuration/last.json 讀成 lastJson
  └─ io.run()
         │
         ▼ （任何一個 signal 或 ReScan 被呼叫）
  propertiesChangedCallback()     500ms debounce；上一輪沒跑完就排下一輪
         │
         ▼
  loadConfigurations()   讀兩個 config 目錄的所有 *.json
         │
         ▼
  PerformScan::run()     每個 config 的 Probe 陣列，
         │               收集所有要查的 D-Bus interface
         ▼
  findDbusObjects()      ObjectMapper GetSubTree 找有哪些 path
  getInterfaces()        每個 path/interface 做 GetAll（重試 5 次）
         │
         ▼ （一個 config 一個 probe，destructor 鏈式跑下一輪 scan）
  doProbe()              評估 probe 表達式（TRUE/FALSE/AND/OR/FOUND/MATCH_ONE）
  probeDbus()+matchProbe()  比對：string 走 regex、數字/其他走相等
         │
         ▼ （probe 通過）
  updateSystemConfiguration()
     ├─ templateCharReplace()   $BUS/$ADDRESS/$index 取代 + 簡單算式
     ├─ generateDeviceName()    名稱重複時用發現順序去重
     └─ applyExposeActions()    Bind*/DisableNode
         │
         ▼ （全部 probe 完 → callback）
  pruneConfiguration()     移除這次沒探到的 device（PowerState=On 的除外）
  publishNewConfiguration()
     ├─ loadOverlays()          new_device 實例化 i2c device、mux 建 symlink
     ├─ writeJsonFiles()        快照寫 /var/configuration/system.json
     └─ postToDbus()            board / item / association 全發上 D-Bus
         │
         ▼
  startRemovedTimer()    10s 後才移除「上次有、這次沒」的 device
```

## fru-device 怎麼運作

主體管「inventory」，`fru-device` 管「每顆 FRU 裡面寫了什麼」。它把 i2c 上所有找得到的 FRU EEPROM 讀回來、解析、掛成 D-Bus 物件。

### 開機（`main()`）

1. 列 `/dev/i2c-*` 拿到所有 i2c bus。
2. 抓 bus name `xyz.openbmc_project.FruDevice`；在 `/xyz/openbmc_project/FruDevice` 掛四個 method：
   - `ReScan()`：全掃一次。
   - `ReScanBus(bus)`：只掃一顆 bus。
   - `GetRawFru` / `WriteFru(bus, addr, data)`：讀原始 FRU / 寫 FRU。
3. **power match**：host 變 `Running` 就全掃一次（很多 FRU 要 host 供電才讀得到）。
4. **inotify**：盯 `/dev`（i2c bus 出現/消失 → 掃該 bus 和它的 root bus）；盯 `<T>/psufru`（virtual PSU FRU 檔出現 → 掃對應 PSU bus）。
5. 開機先全掃一次 + `DoFruConfig()`。

### 掃描（`rescanBusses()` / `rescanOneBus()`）

- **`rescanBusses()`**（全掃，5s debounce，掃前建 `/tmp/fru_scan.lock` 當旗標）：
  1. 清舊的 busmap 和所有 D-Bus 物件。
  2. **平台判斷**（看 marker 檔存在否）：`<T>/TheiaTethys2.0` → PSU 在 bus 7；`<T>/Scorpio` → bus 23；否則當預設平台 → bus 10。
  3. 先跑專用流程（下兩條），把處理過的地址放進 skip 清單：
     - **`processPsuTable()`**：PSU 不走一般掃描。每筆（bus、PSU 地址 0x58、FRU 地址 0x50、virtual FRU 檔）：先要 presence 檔（CPLD 偵測產生的 `<T>/psufru/virtual-psu_N.fru.bin`）存在；probe 兩個地址看在不在；讀 live FRU（含 16-bit EEPROM 偵測），header 壞或讀不到就**用 virtual 檔墊底**——PSU 不會從 inventory 消失。名字 override 成 `PSU_1`/`PSU_2`。
     - **`processCachedFruTable()`**（只跑預設平台）：standby 供電的 FRU（OCP 26:0x50、GPU_0 27:0x53、GPU_1 41:0x53、NVME 46:0x54）。host power cycle 時這些 EEPROM 可能瞬讀不到，所以**優先讀 boot 時 `fxn-first-one` 產生並驗過 checksum 的 cache 檔**（`<T>/ocpfru/`、`<T>/gpufru/`、`<T>/nvmefru/`）；cache 沒有/壞才改讀 sysfs eeprom（重試 3 次、間隔 100ms）。
  4. 一般掃描：逐 bus、逐地址偵測 EEPROM、讀 FRU、驗 header。
  5. 掃完：baseboard FRU 從 `/etc/fru/baseboard.fru.bin` 當 (bus 0, addr 0) 放上去（Tethys liquid / Theia air 平台例外）；每顆 device `addFruObjectToDbus()`。
- **`rescanOneBus()`**：單 bus 版，流程相同（清該 bus 舊物件 → 專用表 filter 到該 bus → 單 bus 掃 → 重發該 bus 的 D-Bus 物件）。
- **`buildNameOverridesTable()`**：固定 (bus, addr) → 名字（GPU_0/GPU_1/OCP/SSD_0..3），讓 D-Bus 物件名穩定。

### 發布（`addFruObjectToDbus()`）

- 解析 FRU bytes 成欄位；路徑 = `/xyz/openbmc_project/FruDevice/<ProductName>`（override 名優先；重名加 `_N`）。
- interface `xyz.openbmc_project.FruDevice`，所有欄位都是 property；另註冊 `BUS`、`ADDRESS`、`MUX`（channel 名）。
- **`PRODUCT_ASSET_TAG` 是 readWrite**：set 時走 `updateFRUProperty()`——在 FRU 的 product info area 找到 asset tag 欄位、改值、修 area length 和 checksum、整顆 FRU 寫回、再 rescan。

### 讀寫與配置

- **`readBaseboardFRU()` / `readFXNFRU()`**：baseboard 從 `/etc/fru/baseboard.fru.bin`、virtual/PSU 從指定檔讀整檔。
- **`writeFRU()`**：先 parse 驗證格式；(0,0) 寫 baseboard 檔；sysfs 有 eeprom 檔就寫 eeprom；否則 i2c smbus 寫。
- **`updateCachedFruIfNeeded()`**：寫到「有 cache 的地址」後，把新 FRU 原子寫進 cache 檔（tmp + rename）——不然隨後的 rescan 會用舊 cache 把記憶體裡的值蓋回去，讓寫入「看起來沒生效」。
- **`DoFruConfig()` / `getFruConfig()`**：讀 `/usr/share/entity-manager/configurations/eeprom.json`，每筆 `FRU_EEPROM` 建一個 `Inventory.Item.FruConfig` 物件（Bus/Address/FruId/FruSize），供查詢用。
- **`loadBlocklist()`**：讀 `/usr/share/entity-manager/blacklist.json`（`{"buses":[3,48,54,55]}`）。code 只認 `{"bus":N,"addresses":[...]}` 結構的 entry，純數字清單實際上不會 block 任何 bus。

## devicetree-vpd-parser

最小的一個：device tree 有 machine context 節點就 `populateFromDeviceTree()` 把 VPD 資料讀進 `MachineContext` 物件，抓 bus name `xyz.openbmc_project.MachineContext` 後進 loop。讓其他程式從 D-Bus 拿到「這顆 BMC 跑在什麼機器上」。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/usr/share/entity-manager/configurations/*.json` | 硬體描述配置（各平台） |
| `/usr/share/entity-manager/configurations/schemas/` | JSON schema（`global.json`、各 type 的驗證檔） |
| `/etc/entity-manager/configurations/` | host 端覆寫配置（優先於上面的） |
| `/var/configuration/system.json` | 每輪發布後的 config 快照（跨開機保留的來源） |
| `/var/configuration/version` | 上次開機時 `/etc/os-release` 的 hash |
| `/tmp/configuration/last.json` | 開機時由 system.json 搬來，「上次開機狀態」 |
| `/etc/fru/baseboard.fru.bin` | baseboard FRU 的檔（當 bus 0 addr 0） |
| `<T>/psufru/virtual-psu_N.fru.bin` | PSU presence + virtual FRU（CPLD 偵測產生） |
| `<T>/ocpfru/`、`<T>/gpufru/`、`<T>/nvmefru/` | standby 供電 FRU 的 boot cache |
| `<T>/TheiaTethys2.0`、`<T>/Scorpio`、`<T>/Cepheus` ... | 平台 marker 檔（fru-device 判平台用） |
| `/dev/i2c-mux/<mux名>/<channel名>` | Mux channel 的 symlink（overlay 建） |
| `/tmp/fru_scan.lock` | FRU 掃描進行中的旗標檔 |
| `/usr/share/entity-manager/blacklist.json` | i2c bus 黑名單（目前格式跟 code 不匹配，實際不生效） |

## 新手常問

- **為什麼有些東西要 host 開機才看得到？** 很多 FRU/感測器要 host 供電。配置裡標 `PowerState: "On"` 的 device 在 host 關機時不 prune，開機後才重新判定。
- **為什麼我 runtime 改的值重開機還在？** 每次發布都存 `/var/configuration/system.json`；開機時韌體版本沒換（`/etc/os-release` hash 同）就把它恢復成 lastJson，merge 時舊記錄優先。
- **為什麼 device 消失要等 10 秒才移除？** `startRemovedTimer()` 的寬限期，避免 power cycle 期間短暫讀不到就亂刪。
- **同名 board 會怎樣？** 第二個直接被忽略（vendor crash fix）；同 board 內重名 item 用發現順序數字去重。
- **要加一顆新硬體怎麼做？** 在配置目錄寫一筆 JSON：`Name`/`Type`/`Probe`（判斷它在不在的 D-Bus 條件）/`Exposes`（要公佈哪些子項目、bus、地址），重建後 entity-manager 下次重掃就會處理它。
- **blacklist.json 為什麼不生效？** 目前是純數字 bus 清單（`{"buses":[3,48,54,55]}`），但 code 只解析 `{"bus":N,"addresses":[...]}` 物件格式的 entry，兩種格式對不上。
