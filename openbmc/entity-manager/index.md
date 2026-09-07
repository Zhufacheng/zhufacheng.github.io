# entity-manager

OpenBMC entity-manager 的 vendor fork，在 onetree 树的 vendor layer `recipes-phosphor/configuration/entity-manager/` 下（完整 source 在同目錄的 fork 目錄裡）。一個 recipe 建出三個 daemon：

- **`entity-manager`**：主體。讀 hardware description JSON，跟 D-Bus 上實際出現的物件比對，把「偵測到的硬體」發布成 system inventory。
- **`fru-device`**：掃 i2c 上的 FRU EEPROM（外加幾組固定地址），解析後發布到 D-Bus。
- **`devicetree-vpd-parser`**：從 device tree 讀 machine context 發布到 D-Bus。

> 下文 `<T>` 代表 /tmp 下的一個 vendor 子目錄（確切名稱見 code）。

## 流程圖（entity-manager 主體）

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
     └─ applyExposeActions()    Bind* / DisableNode
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

## 重要 function（依執行順序）

### 起程式

- **`main()`**：抓 bus name `xyz.openbmc_project.EntityManager`；object manager 只管 `/xyz/openbmc_project/inventory` 子樹；註冊 `NameOwnerChanged`（任何 service 上下線都重掃）、`InterfacesAdded/Removed`（payload 含 probe 用到的 interface 才重掃）；`entityIface` 掛 `ReScan` method；最後 `io.run()`。
- **`setupPowerMatch()`**：match `/xyz/openbmc_project/state/host0` 的 `CurrentHostState` 變化，state 結尾是 `Running` 就記 power on；開機先 `Get` 一次現值。
- **`fwVersionIsSame()`**：把 `/etc/os-release` 全文 hash，跟 `/var/configuration/version` 存的比。一樣（韌體沒換）或 `/var/configuration/system.json` 存在，就把 system.json copy 到 `/tmp/configuration/last.json`、刪原檔、parse 進 `lastJson`（當「上次開機狀態」用）；否則清掉舊 snapshot。

### 訊號觸發與防抖

- **`propertiesChangedCallback()`**：所有重掃的入口。500ms timer 防抖（被 abort 代表有更新的請求）；`inProgress` 鎖，上一輪還在跑就把這輪再排一次。流程：copy 舊 config → `loadConfigurations` → 起 `PerformScan`，scan 完的 callback 做 prune + publish。
- **`iaContainsProbeInterface()` / `irContainsProbeInterface()`**：解 InterfacesAdded/Removed 的 payload，看新出現/消失的 interface 有沒有跟 `getProbeInterfaces()` 收集到的集合相交。
- **`getProbeInterfaces()`**：掃所有 config JSON 的 Probe 字串，把 `(` 前面的 interface 名收集起來（開機算一次，給上面兩個 match 用）。

### 讀設定

- **`loadConfigurations()`**：讀 `PACKAGE_DIR/configurations`（=`/usr/share/entity-manager/configurations`，bbappend 把各平台 JSON 裝在這）和 `/etc/entity-manager/configurations` 兩目錄的所有 `*.json`；array 就拆成多筆。schema 驗證（valijson + `global.json`）目前註解掉。

### 掃描（PerformScan）

- **`PerformScan::run()`**：把每筆 config 的 `Probe`（string 或 string array）建一個 `PerformProbe`；同時從 probe 字串抽出要查的 D-Bus interface，全部丟給 `findDbusObjects`。缺 `Probe` 或 `Name` 的 config 直接丟掉。
- **`findDbusObjects()`**：對 ObjectMapper 做 `GetSubTree("/", depth 0, interfaces)`，一次拿回「哪些 path 暴露這些 interface」；失敗重試 5 次（每次 10s），`ENOENT` 代表 mapper 沒找到就回。
- **`processDbusObjects()` / `registerCallback()`**：對每個找到的 path 註冊 `PropertiesChanged` match（該 path 上 property 一改就重掃）；對每個非 `org.freedesktop.*` 的 interface 排 `getInterfaces`。
- **`getInterfaces()`**：`org.freedesktop.DBus.Properties.GetAll` 抓每個 interface 的所有 property，存進 `scan->dbusProbeObjects[path][interface]`；失敗 2s 後重試、最多 5 次。

### Probe（PerformProbe）

- **`findProbeType()`**：probe 字串含 `TRUE/FALSE/AND/OR/FOUND/MATCH_ONE` 其中一個就是控制詞，否則當 D-Bus probe。
- **`doProbe()`**：依序評估 probe 陣列。`TRUE/FALSE` 直接定值；`FOUND('name')` 看之前哪個 probe 通過過；D-Bus probe 拆成 interface 名（`(` 前）+ JSON 條件（`(...)` 內），丟 `probeDbus`；`AND/OR` 跟前一個結果合併；`MATCH_ONE` 最後只留最後一個 match 的 device。
- **`probeDbus()`**：遍歷 `dbusProbeObjects`，有該 interface 的 path 逐筆比對；所有列出的 property 都要 match 才算中，中的 path+properties 記進 `foundDevs`。
- **`matchProbe()`**：string 對 string 用 regex search；其他型別直接相等。
- **`PerformProbe` destructor**：`doProbe` 通過就呼叫 `scan->updateSystemConfiguration`（把這筆 config 套進系統配置）。
- **`PerformScan` destructor**：有 probe 通過就複製一份新 `PerformScan`（帶著已通過的 probe 名單和已抓的 D-Bus 資料）再跑一次——所以「FOUND」才能引用先前的 probe；全部處理完才呼叫最終 callback。

### 合併進 system configuration

- **`updateSystemConfiguration()`**：先處理「上次就有」的 device（recordName 在現有 config 或 `lastJson` 裡）：原樣保留（D-Bus  runtime 改過的值不丟）、從 missing 清單刪掉。再對每個新發現的 device：套 template、記 `FoundProbePath`、套 expose actions，寫進 `systemConfiguration[recordName]`。
- **`getRecordName()`**：probe 名 + 該 device 的 interface 內容 hash，當這筆 device 的 key。
- **`generateDeviceName()`**：`Name` 當 template 做取代；若取代後跟已用的名字重名，用「第幾個發現的」數字再取代一次去重。
- **`templateCharReplace()`**：string 裡的 `$index`、`$PROP` 換成該 D-Bus object 的 property 值；`$PROP+10` 這類可以接著算式（`+ - % * /`，走 `expression::evaluate`）；整串就是一個 `$PROP` 時直接換成原始型別；`0x` 開頭的數值字串轉成數字。
- **`applyExposeActions()`**：expose 裡的 `BindXXX: "Name"` / `BindXXX: ["Name", ...]` 把別筆 config 的 expose 抓過來嵌進來（Status=okay）；`DisableNode` 把目标 expose 設 `Status: "disabled"`。

### 發布到 D-Bus（postToDbus）

- **`postToDbus()`**：對每個 board：
  - path = `/xyz/openbmc_project/inventory/system/<boardType 小寫>/<Name>`（非法字元換 `_`）；
  - 同名 board 直接忽略（duplicate board crash fix）；
  - 建 `Inventory.Item` + `Inventory.Item.<Type>` 兩個 interface，註冊 `AddObject` method；
  - board 層的 object 型 property 各建一個 interface；其中 `Inventory.Decorator.AssetTag` 是 **readWrite**（並記下該 board 的 `FoundProbePath` 供寫 asset tag 用）；
  - 每個 `Exposes` item（`Status=disabled` 跳過）：path 接 `/<ItemName>`，建 `Configuration.<Type>` interface；`BMC`/`System` 另加對應 Inventory interface；item 內的 object 再建 `Configuration.<Type>.<Prop>`，array 建 `...<Prop><0..n>`；
  - 最後 `topology.getAssocs()` 的 result 註冊成每個 board 的 `Association.Definitions.Associations`。
  - 取值一律用 `systemConfiguration`（不是原始 JSON），這樣 D-Bus runtime 改過的值不会被下次 publish 覆蓋。
- **`getPermission()`**：interface 名在 `FanProfile, Pid, Pid.Zone, Stepwise, Thresholds, Polling, Preserve` 七種之内的 property 是 readWrite，其他 readOnly。
- **`createInterface()` / `populateInterfaceFromJson()`**：建 interface 並逐 property 註冊（型別跟隨 JSON；readWrite 的數字統一 double）；readWrite 的 interface 同時掛 `Delete` method（把該 config 設 null、寫 snapshot、移除 interface）。
- **`createAddObjectMethod()`**：board 上的 `AddObject`：傳入 `{Name, Type, ...}`，照 `schemas/<type>.json` 驗證，填進 Exposes 的 null 槽或 append，寫 snapshot，再建對應 `Configuration.<Type>` interface（readWrite）。
- **`Topology`**：`addBoard` 記 `DownstreamPort`（依 `ConnectsToType` 分組、`PowerPort` 標記）和 upstream port（Type 結尾是 `Port`）；`getAssocs` 配對後，downstream board 得 `contained_by/containing` 關聯、有 `PowerPort` 的再給 upstream 加 `powered_by/powering`；`remove` 把該 board 從兩邊清掉。

### 移除 device

- **`deviceRequiresPowerOn()`**：config 有 `PowerState: "On"/"BiosPost"` 代表這 device 只有 host 開機才看得到。
- **`pruneConfiguration()`**（scan callback 裡）：這輪沒探到、又不在 lastJson 保留集的 device，移除它的所有 D-Bus interface、清 boardNames、`topology.remove`。host 沒開機時，`PowerState=On` 的 device 不移（不知道是沒了還是沒開機）。
- **`pruneDevice()` / `startRemovedTimer()`**：10s timer（只在「上輪 power off、這輪還 off」或「第一次 power on 前」生效），把「`lastJson` 有、新 config 沒有」的 device 移除——避免 power cycle 期間短暫看不到就亂刪。

### 持久化與 overlay

- **`writeJsonFiles()`**：`systemConfiguration` 整個 dump 到 `/var/configuration/system.json`（每輪 publish 都寫；也是下次開機 lastJson 的來源）。
- **`loadOverlays()` / `exportDevice()` / `buildDevice()`**：每筆 expose 的 `Type` 若在 `devices::exportTemplates` 白名單（EEPROM、TMP75、Mux 等），就寫 `/sys/bus/i2c/devices/i2c-<Bus>/new_device` 實例化 kernel driver（參數是 `$Address` 取代後的字串）；有 `hwmon` 要求的 device 要驗到 `hwmon` 子目錄才算成功，失敗就寫 `delete_device`、500ms 後重試、最多 5 次。
- **`linkMux()`**：Mux 型 device 建好後，把 `/sys/.../channel-N` 指向的 bus 建 symlink 到 `/dev/i2c-mux/<muxName>/<channelName>`，讓設定可以用 channel 名當 bus。

## fru-device

### 起程式

- **`main()`**：列 `/dev/i2c-*`；`loadBlocklist`；抓 bus name `xyz.openbmc_project.FruDevice`；`/xyz/openbmc_project/FruDevice` 上掛 `ReScan`、`ReScanBus(bus)`、`GetRawFru`、`WriteFru(bus, addr, data)`；power match（host 變 `Running` 就全掃一次）；inotify 盯 `/dev`（i2c bus 出現/消失 → 掃該 bus 和它的 root bus）和 `<T>/psufru`（virtual PSU FRU 檔出現 → 掃對應 PSU bus）；開機先 `rescanBusses` + `DoFruConfig`。

### 掃描

- **`rescanBusses()`**：5s debounce；建 `/tmp/fru_scan.lock`（掃完刪）；清舊 busmap 和所有 D-Bus interface；平台判斷（看 marker 檔）：`<T>/TheiaTethys2.0` → PSU 在 bus 7、`<T>/Scorpio` → bus 23、否則當 Cepheus → bus 10；先 `processPsuTable`（PSU 專用流程）、Cepheus 再 `processCachedFruTable`（standby 供電 FRU），兩者把地址放進 skip 清單；然後一般掃描（逐 bus、逐 address 偵測 EEPROM、讀 FRU、驗 header）；掃完 callback：baseboard FRU 從 `/etc/fru/baseboard.fru.bin` 當 bus 0 addr 0 放上去（Tethys liquid / Theia air 平台例外）、每個 device `addFruObjectToDbus`。
- **`rescanOneBus()`**：單 bus 版。清該 bus 的舊 D-Bus 物件和 busmap，同樣先跑 PSU/cached 表（filter 到該 bus）再單 bus 掃，掃完重發該 bus 的 D-Bus 物件。
- **`processPsuTable()`**：PSU 不走一般掃瞄。每筆（bus、PSU 地址 0x58、FRU 地址 0x50、virtual FRU 檔）：先要 presence 檔（`<T>/psufru/virtual-psu_N.fru.bin`）存在；probe 0x58/0x50 看在不在；都在/任一在就讀 live FRU（含 16-bit 偵測），header 壞就用 virtual 檔；結果放進 busmap（key 用 PSU 地址）、名字 override 成 `PSU_1/PSU_2`。
- **`processCachedFruTable()`**：Cepheus 的 standby 供電 FRU（OCP 26:0x50、GPU_0 27:0x53、GPU_1 41:0x53、NVME 46:0x54）。先讀 boot 時 `fxn-first-one` 產生並驗證過 checksum 的 cache 檔（`<T>/ocpfru/`、`<T>/gpufru/`、`<T>/nvmefru/`）；cache 沒有/壞才改讀 sysfs eeprom（重試 3 次、間隔 100ms）。目的：host power cycle 期間 i2c mux 瞬斷不會把 FRU 從 inventory 掉下去。
- **`buildNameOverridesTable()`**：固定 (bus, addr) → 名字：GPU_0/GPU_1/OCP/SSD_0..3，讓 D-Bus 物件名穩定。
- **`addFruObjectToDbus()`**：parse FRU bytes 成欄位；path = `/xyz/openbmc_project/FruDevice/<ProductName>`（override 名優先；重名加 `_N`）；interface `xyz.openbmc_project.FruDevice` 登記所有欄位；`PRODUCT_ASSET_TAG` 是 readWrite（set 時走 `updateFRUProperty` 寫回 FRU 再 rescan）；另註冊 `BUS`、`ADDRESS`、`MUX`（channel 名）。
- **`loadBlocklist()`**：讀 `/usr/share/entity-manager/blacklist.json`（`{"buses":[3,48,54,55]}`）。code 只認 `{"bus":N,"addresses":[...]}` 結構的 entry，純數字清單實際上不會 block 任何 bus。

### 讀寫 FRU

- **`readBaseboardFRU()` / `readFXNFRU()`**：baseboard 從 `/etc/fru/baseboard.fru.bin`、virtual/PSU 從指定路徑讀整檔。
- **`updateCachedFruIfNeeded()`**：`WriteFru` 成功寫到「有 cache 的 address」後，把新 FRU 原子寫進 cache 檔（tmp+rename）——不然 rescan 會用舊 cache 把記憶體裡的值蓋回去，讓寫入「看起來沒生效」。
- **`writeFRU()`**：先 parse 驗證 FRU 格式；(0,0) 寫 baseboard 檔；sysfs 有 eeprom 檔就寫 eeprom；否則 i2c smbus 寫。
- **`updateFRUProperty()`**：在 product info area 找 asset tag 欄位（依 type/length code 走）、改值、修 area length 和 checksum、整顆 FRU 寫回。
- **`getFruConfig()`**：向 ObjectMapper 查 `Inventory.Item.FruConfig` 物件，依 (bus, addr) 回 `FruId/FruSize`。
- **`DoFruConfig()`**：讀 `/usr/share/entity-manager/configurations/eeprom.json`，每筆 `FRU_EEPROM` 建一個 `/xyz/openbmc_project/FruDevice/<Name>` 物件（`Bus/Address/FruId/FruSize`），供上面查詢。

## devicetree-vpd-parser

- **`main()`**：device tree 有 machine context 節點就 `populateFromDeviceTree()` 把 VPD 資料讀進 `MachineContext` 物件，抓 bus name `xyz.openbmc_project.MachineContext` 後進 loop。

## vendor patches（patches/series 共 17 個）

| patch | 內容 |
|---|---|
| Entity-manager: Add support to update assetTag | asset tag 可經 D-Bus 寫回 FRU |
| fru-device: Add MUX channel name to FRU objects | FRU 物件帶 MUX channel 名 |
| Add Config-FRU Support | `eeprom.json` → `FruConfig` D-Bus 物件 |
| Add logs to fwVersionIsSame | 版本 hash 判斷加 log |
| Add condition for journal error/info handling | journal log 分級處理 |
| Coverity fix | static analysis 修正 |
| Converting journal log into dbus | log 轉 D-Bus |
| Add new interface for partial preserve config support | `Preserve` interface：部分 config 可 runtime 改且保留 |
| Adding MUX and Drives present in HSBP in json config | HSBP 設定加 MUX、disk presence |
| Added Fix For SDR-Perceive-Configuration | SDR preserve 修正 |
| Added Test-Fru Feature | `ENABLE_TEST_FRU` 測試用 FRU 物件 |
| dynamic threshold configuration for SOLUM PSU | PSU 動態 threshold |
| Change HSBP FRU address and add MUX mode configuration | HSBP FRU 地址、MUX mode 設定 |
| entity-manager: fix crash from duplicated boards | 同名 board 直接忽略，不再 crash |
| Added fix for SDR-Fail | SDR fail 修正 |
| FruRescan-SDR-Fix | FRU rescan 對 SDR 的修正 |
| Reverting community patch for Preserve config failure | 回退 community 的 preserve 失敗處理 |

## 幾個重點

- **inventory 路徑**：board = `/xyz/openbmc_project/inventory/system/<type>/<Name>`，item 再接一層；FRU = `/xyz/openbmc_project/FruDevice/<ProductName>`。
- **可 runtime 改的 interface**：`FanProfile, Pid, Pid.Zone, Stepwise, Thresholds, Polling, Preserve` 七種是 readWrite——PID/Thresholds 參數就是靠這路徑熱改，改完寫進 `/var/configuration/system.json` 快照，重開機由 last.json 還原。
- **state 跨開機保留**：每次 publish 都存快照；開機時韌體版本沒換（`/etc/os-release` hash 同）就把上次的 config 搬成 lastJson 恢復，D-Bus 改過的值（asset tag、PID 參數...）不會因為重新 probe 被蓋掉。
- **power off 期間不乱刪**：`PowerState: "On"` 的 device 在 host 關機時不 prune；「上次有這次沒」的 device 要過 10s timer 才移除。
- **同名 board 直接忽略**（crash fix）；同 board 內重名 item 用發現順序數字去重。
- **平台判定靠 marker 檔**：fru-device 依 `<T>/TheiaTethys2.0`、`<T>/Scorpio`、`<T>/Cepheus` 等 marker 檔決定 PSU bus（7/23/10）和要不要跑 cached FRU 表。
- **PSU 用 virtual FRU 墊底**：presence 檔（CPLD 偵測產生的 `<T>/psufru/virtual-psu_N.fru.bin`）存在但 live FRU 讀不到時，用 virtual 檔，PSU 不會從 inventory 消失。
- **standby 供電 FRU 走 cache-first**：OCP/GPU/NVME 的 EEPROM 在 host power cycle 時可能瞬讀不到，所以優先用 `fxn-first-one` boot 時驗證過的 cache；`WriteFru` 之後會同步刷新 cache 避免舊值蓋新值。
- **overlay 只負責「把 i2c device 建出來」**：kernel driver 實例化（new_device）+ mux channel symlink；sensor 本身由 phosphor-sensor 看 hwmon 處理。
- **blacklist.json 目前形同虛設**：格式（純數字 bus 清單）跟 code 預期（`{bus, addresses}` 物件）不匹配，沒有 bus 真的被 block。
