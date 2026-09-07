# Redfish

Redfish 是 DMTF 的新一代 out-of-band 管理 API：HTTP(S) + JSON、REST 式資源模型、schema 化。OpenBMC 由 `bmcweb` 實作 Redfish，是目前預設推薦的管理介面（IPMI 定位為 legacy 相容介面）。

## 背景概念

- **ServiceRoot**：`/redfish/v1/` 是入口，列出所有 top-level 資源。
- **資源（Resource）**：每個 Redfish 物件是一個有定義 schema 的 JSON 文件（Chassis、Manager、ComputerSystem 等），資源之間以 `@odata.id` 連結互相導覽。
- **D-Bus 後端**：`bmcweb` 自己**不保存任何狀態**——所有資料即時從其他 daemon 的 D-Bus 查詢，寫入操作（power、account、update 等）也是翻譯成 D-Bus call。bmcweb 的角色 = 協定翻譯層。
- **Session / token**：兩種認證：basic auth 或 session login，成功後拿到 `X-Auth-Token`（JWT token）；權限由 BMC user 的 role（Administrator / Operator / ReadOnly）映射。

## 怎麼運作

1. **啟動**：`bmcweb` 監聽 TCP 443（HTTP 80 redirect 到 HTTPS）。TLS 憑證預設是 first boot 時產生的 self-signed（有專責的 cert setup 服務）。
2. **請求映射**：每個 URL path（例如 `/redfish/v1/Chassis/1/Thermal`）對應 `bmcweb` 裡一個 handler；handler 通常：
   - 用 `xyz.openbmc_project.ObjectMapper`（`GetSubTree`）解析目標 D-Bus 物件；
   - 讀 property（sensors、state、version…）；
   - 依 Redfish schema 轉成 JSON 資源回傳。
3. **寫入操作**：例如 `POST /redfish/v1/Managers/1/Actions/Manager.Reset` → `bmcweb` 翻譯成對應 daemon 的 D-Bus method / property set。長操作（firmware update）回一個 Task 資源，client 輪詢 Task 看進度。
4. **事件**：EventService 支援 subscription（HTTP callback 或 websocket）；`bmcweb` 訂閱 D-Bus signal，轉成 Redfish event 送出。

典型資源樹：

```
/redfish/v1/
├── Chassis/          power state、Thermal（sensors）、Drives、NetworkInterfaces
├── Systems/          host 系統（Bios、Boot、LogServices）
├── Managers/         BMC 本身（FirmwareVersion、NetworkProtocol、Reset 等）
├── UpdateService/    firmware update 入口（UpdateTask、FirmwareInventory）
├── LogService/       log entries（journal、SEL）
├── TaskService/      長任務狀態
├── SessionService/   session / token
├── AccountService/   user 與 role
└── EventService/     事件訂閱
```

D-Bus → Redfish 映射示例：

| Redfish 資源 | D-Bus 來源 |
|---|---|
| Chassis / PowerState | `xyz.openbmc_project.State.Host`（`CurrentHostState`） |
| Chassis / Thermal | `/xyz/openbmc_project/sensors/...` 下的 `Sensor.Value` + `Sensor.Threshold.*` |
| Chassis / Drives、PCIe 設備 | `/xyz/openbmc_project/inventory/...` 下的 inventory 物件樹 |
| Manager（版本等） | `/xyz/openbmc_project/Software/...` 下的 `Software.Version` 系 interface |
| LogService / LogEntries | `phosphor-logging` / `phosphor-sel-logger` expose 的 D-Bus 物件 |
| UpdateService | `/xyz/openbmc_project/Software/...` 下的 update / activation interface（含 `Software.Activation` 進度，見 host-firmware-update 頁） |

認證流程：

```
POST /redfish/v1/SessionService/Sessions（username + password）
   │  對照 phosphor-user-manager 管理的 user 資料庫
   ▼
201 Created + X-Auth-Token（JWT）+ session 物件
   │
後續請求帶 X-Auth-Token（或直接 basic auth）
   │  bmcweb 驗 token、role → 決定讀寫權限
   ▼
handler 執行 → 200 JSON
```

## 相關專案 / daemon

- `bmcweb`：Redfish server 本體（C++，boost::asio + sdbusplus）。
- `phosphor-state-manager`：power state 資料來源。
- `phosphor-logging` / `phosphor-sel-logger`：LogService 資料來源。
- `phosphor-software-manager`：UpdateService 執行後端。
- `phosphor-user-manager`：Account 與 session 認證的 user/role 資料來源。
- 各 sensor daemon（`phosphor-sensors`、`phosphor-virtual-sensor` 等）：Thermal 資料來源。
- `entity-manager`：硬體描述，間接決定 inventory 形態。

## D-Bus

`bmcweb` 是 D-Bus 的 consumer，主要依賴：

- `xyz.openbmc_project.ObjectMapper` — 物件發現（`GetSubTree`）
- `xyz.openbmc_project.Sensor.Value` / `Sensor.Threshold.*` — sensors
- `xyz.openbmc_project.State.Host` — power state
- `xyz.openbmc_project.Software.Version` / `Software.Activation` — firmware 版本與 update 進度
- `phosphor-user-manager` 提供的 user interface

具體物件 path 隨資源而異，`bmcweb` 用 ObjectMapper 動態解析，不在程式碼裡 hardcode。

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/etc/ssl/certs/https/server_cert.pem`、`server_key.pem` | `bmcweb` 的 HTTPS 憑證（first boot self-signed，可替換） |

## 新手常問

- **Redfish 沒網路能用嗎？** 不能，它是 HTTP API，要經過 BMC 的網路介面；host 端無網路情境還有 KCS（IPMI）可用。
- **憑證從哪來、要怎麼換？** 預設 first boot 自動產生 self-signed；可直接換檔案（重啟 `bmcweb` 生效）或用 Manager 的憑證管理介面。
- **401 / 403 各代表什麼？** 401 = token 失效或未認證；403 = 該 user role（例如 ReadOnly）沒權限做這操作。
- **為什麼某資源是空的？** 對應 daemon 沒跑或物件還沒上線（例如 inventory 空，查 `entity-manager` 與 probe daemon）。
- **UpdateService 和 TaskService 的關係？** 長 update 請求先回一個 Task 資源；client 輪詢 Task 拿進度（背後的 `Software.Activation` progress 來自 D-Bus）。
