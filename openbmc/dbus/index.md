# D-Bus

D-Bus 是 OpenBMC 的中樞 IPC：所有 daemon 把自己的狀態、設定、事件以 D-Bus 物件發佈在 system bus 上，其他 daemon 當 consumer 來讀取或訂閱。整個系統沒有「中央設定中心」——daemon 之間靠 D-Bus 物件模型互相發現、互相掛勾。sensor、log、host state、firmware update 等所有功能都建在它上面。

## 背景概念

- **system bus**：D-Bus 系統匯流排，broker 是 `dbus-daemon`。daemon 連上後取得 **well-known name**（如 `xyz.openbmc_project.EntityManager`），再發佈物件、呼叫方法。
- **物件模型**：一個 D-Bus 物件 = **object path**（如 `/xyz/openbmc_project/sensors/temperature/P1_INLET`）+ **interface**（如 `xyz.openbmc_project.Sensor.Value`）+ **properties / methods / signals**。同一個 path 可以掛多個 interface。
- **Object Manager**：實作 `org.freedesktop.DBus.ObjectManager` 的物件管一棵子樹：`GetManagedObjects` 一次拿回子樹內所有物件，物件增刪時發 `InterfacesAdded` / `InterfacesRemoved` 訊號。OpenBMC 的 daemon 通常各自管一棵子樹（例如 sensor daemon 管 `/xyz/openbmc_project/sensors` 整棵樹）。
- **命名慣例**：path 與 interface 一律在 `xyz.openbmc_project.*` 之下；path 形如 `/xyz/openbmc_project/<category>/<instance>`。
- **標準 interface**：`org.freedesktop.DBus.Properties`（Get/GetAll/Set + `PropertiesChanged` 訊號）、`org.freedesktop.DBus.ObjectManager`、`org.freedesktop.DBus.Introspectable`、`org.freedesktop.DBus.Peer`。

## 怎麼運作

### Provider：daemon 怎麼暴露物件

1. 連上 system bus、`request_name("xyz.openbmc_project.<Service>")`。unit 通常是 `Type=dbus`，所以「拿到 name」= 服務啟動成功。
2. 用 `sdbusplus`（C++ binding）在 object manager 下建立物件：`add_interface(path, interface)`、填 properties、declare methods 與 signals。
3. property 改變 → 發 `PropertiesChanged`；新物件 → 發 `InterfacesAdded`。consumer 是被 push 通知，不用 poll。

### Consumer：daemon 怎麼用別人的物件

1. **目標已知**：直接向該 well-known name 做 async 呼叫（讀 property、叫 method）。
2. **發現（目標未知）**：問 `xyz.openbmc_project.ObjectMapper`：
   - `GetSubTree(path, depth, interfaces[])`：找出子樹下符合條件的所有物件，回傳每個物件的 path、interface 清單、以及 owner service。
   - `GetObject(path, interfaces[])`：查某 path 上的指定 interface 歸哪個 service。
3. **事件訂閱**：在 bus 上掛 match rule（例如某 path/interface 的 `PropertiesChanged`、`InterfacesAdded/Removed`、`NameOwnerChanged`），由 callback 接收事件。

### 典型互動

```
provider (sensor daemon)                      consumer (bmcweb / alarm daemon)
  request_name("xyz.openbmc_project.<Service>")
  object manager 管 /xyz/openbmc_project/sensors
  ├─ path: .../sensors/temperature/P1_INLET
  ├─ interface: xyz.openbmc_project.Sensor.Value (Value/Unit/...)
  └─ property 改變 → PropertiesChanged
        │
        ▼   system bus (dbus-daemon)
        │
  consumer:
  ├─ 訂閱 PropertiesChanged (match rule)
  ├─ GetObject / GetSubTree 問 ObjectMapper 找 owner
  └─ async_method_call 讀 property / 叫 method
```

### ObjectMapper

- service name `xyz.openbmc_project.ObjectMapper`，object path `/xyz/openbmc_project/object_mapper`。
- 維護「path + interface → owning service」的索引，讓 consumer 不必預知某個物件是哪個 daemon 提供的。
- 實作在 `phosphor-objmgr` repo。

## 相關專案 / daemon

- `dbus-daemon` — broker。
- `sdbusplus` — C++ binding（asio-based，async 呼叫 + signal 訂閱），OpenBMC C++ daemon 的慣用標準。
- `phosphor-dbus-interfaces` — OpenBMC 標準 interface 的定義（introspection XML + JSON）與產生的 C++ code；新 interface 先加在這裡。
- `phosphor-objmgr` — C++ ObjectManager 輔助函式庫（從 C++ 結構自動產生 D-Bus 物件）+ ObjectMapper service。
- `entity-manager`、`phosphor-logging`、`phosphor-state-manager`、`bmcweb`、`hostipmi` — 典型 producer / consumer。

## D-Bus

well-known name 舉例：

| Name | 提供者 |
|---|---|
| `xyz.openbmc_project.ObjectMapper` | object mapper |
| `xyz.openbmc_project.EntityManager` | `entity-manager` |
| `xyz.openbmc_project.IPMI` | `hostipmi`（D-Bus IPMI interface） |
| `xyz.openbmc_project.bmcweb` | `bmcweb` |

ObjectMapper：path `/xyz/openbmc_project/object_mapper`、interface `xyz.openbmc_project.ObjectMapper`、methods `GetSubTree` / `GetObject`。

常用 object path：

- `/xyz/openbmc_project/sensors/<type>/<name>` — 感測器
- `/xyz/openbmc_project/inventory/system/...` — 硬體 inventory
- `/xyz/openbmc_project/state/host0` — host 狀態
- `/xyz/openbmc_project/Logging/entry/<id>` — log entry

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/run/dbus/system_bus_socket` | system bus 的 socket |
| `/etc/dbus-1/system.d/` | 各 service 的 D-Bus policy（XML，決定誰可以叫什麼） |
| `phosphor-dbus-interfaces` repo（`xml/`、`json/`） | interface 定義與產生的 code |

## 新手常問

- **怎麼看一個 daemon 的物件？** `busctl tree <name>`、`busctl introspect <name> <path>`；用 `dbus-monitor` 看即時訊息。
- **怎麼查一個物件歸誰管？** 對 `xyz.openbmc_project.ObjectMapper` 下 `GetObject` / `GetSubTree`。
- **daemon 掛掉會怎樣？** 它的物件消失、bus 上發 `NameOwnerChanged`；consumer 應該訂閱這個事件並自行處理（例如把值標成不合法）。
- **為什麼不直接讀檔案？** 執行期狀態（sensor、alarm）是動態的、事件需要 push；D-Bus 提供統一的發現、訂閱與權限控制。
- **新 interface 該加在哪裡？** 先加進 `phosphor-dbus-interfaces`（XML/JSON 定義），兩邊用產生的 code，避免各自手寫 interface 定義造成不一致。
