# phosphor-user-manager

`phosphor-user-manager` 是 OpenBMC 的「使用者帳號」daemon：把 BMC 上的 user account（帳號、密碼、權限等級、啟用/停用）集中管在 D-Bus 上，並同步到本機 `/etc/passwd`、`/etc/shadow`。IPMI、Redfish、SSH 登入、KVM / SOL 等所有需要「誰可以做什麼」的機制，授權來源都是它。

## 背景概念

- **權限等級（privilege level）**：每個 user 有一個等級，決定能用哪些介面。等級是 enum：`None=0`、`User=1`、`Operator=2`、`Administrator=3`、`Oem=4`、`NoAccess=255`。IPMI 的 privilege（User / Operator / Administrator，對應 IPMI 等級 2/3/4）與 Redfish 的角色（User / Operator / Administrator）都映射到同一組等級。
- **PAM**：OpenBMC 的 local login（SSH、serial console）走 PAM，最終驗的是 `/etc/shadow` 裡的 password hash——也就是 user-manager 寫入的那份資料。所以「D-Bus 上的帳號」與「本機帳號」是同一份資料的兩個視圖。
- **`Password` property**：是 hash（跟 shadow 同格式），不是明文；set 的時候 daemon 自己 hash。

## 怎麼運作

### 帳號生命週期

1. daemon 開機掃 `/etc/passwd`，把每個 account 建成一個 D-Bus 物件（路徑 `/xyz/openbmc_project/user/<name>`），並持續監聽 passwd/shadow 的變化，讓兩邊視圖同步。
2. **新增**：呼叫 manager 的 `AddUser(name, password, privilege)` → 寫 `/etc/passwd` + `/etc/shadow`（hash 密碼）→ 物件自動上 D-Bus。
3. **修改**（密碼 / 權限 / 停用）：set `User.Attributes` 的 property，daemon 同步回 `/etc/*`。
4. **刪除**：`DeleteUser(name)`。
5. **預設帳號**：首次啟動若系統沒有 user，會建兩個預設帳號（`root` = Administrator、`taobao` = Operator）。

```
client
 │  (IPMI session / Redfish / SSH / KVM)
 ▼
各協定 stack (phosphor-ipmi-net / bmcweb / sshd)
 │  查帳號 + 檢查 privilege
 ▼
phosphor-user-manager (D-Bus)
 │  雙向同步
 ▼
/etc/passwd, /etc/shadow   ← PAM 也讀這份（local login）
```

- **IPMI（`phosphor-ipmi-net`）**：建立 RMCP+ session 時，用 D-Bus 上該 user 的 password 做 RSA challenge/response 認證，並把 session 綁定到該 user 的 privilege level；之後每個 IPMI command 都檢查「session 的 privilege 夠不夠」。
- **Redfish（`bmcweb`）**：`/redfish/v1/AccountService/Accounts` 的 CRUD 直接轉成 user-manager 的 D-Bus 呼叫；每個 Redfish 請求的授權是「session 的 privilege 對照該 endpoint 需要的等級」。
- **local login（SSH / console）**：走 PAM → 驗 `/etc/shadow`；user 被停用（`Enable=false`）時 PAM 拒絕。

## 相關專案 / daemon

- `phosphor-user-manager`：本頁主角，管帳號。
- `phosphor-ipmi-net`：IPMI over LAN（RMCP+），session 認證與 privilege 檢查的消費方。
- `bmcweb`：Redfish，AccountService 與授權的消費方。
- sshd / PAM stack：local login 路徑，最終讀 `/etc/shadow`。

## D-Bus

- Bus name：`xyz.openbmc_project.User.Manager`。
- `/xyz/openbmc_project/user`：manager 物件，interface `xyz.openbmc_project.User.Manager`，methods `AddUser(name, password, privilege)`、`DeleteUser(name)`。
- `/xyz/openbmc_project/user/<name>`：每個帳號一個物件，interface `xyz.openbmc_project.User.Attributes`：

| property | 型別 | 用途 |
|---|---|---|
| `Username` | string（RO） | 帳號名 |
| `Password` | string（RW） | shadow hash；set = 改密碼 |
| `Privilege` | enum（uint8） | 權限等級 |
| `Enable` | bool（RW） | false = 停用，所有登入 / session 被拒 |

## 重要檔案與目錄

| 路徑 | 用途 |
|---|---|
| `/etc/passwd`、`/etc/shadow` | 帳號與 password hash 的實際存儲（PAM 也讀這份） |
| `/etc/pam.d/` | local login（SSH、console）的 PAM 配置 |

## 新手常問

- **改 D-Bus 的 password 跟用 `passwd` 改有什麼差別？** 沒有差別，同一份資料：user-manager 會監聽 `/etc/passwd`、`/etc/shadow` 的變化反向同步，兩邊改都會反映到對方。
- **`Enable=false` 和刪帳號差別在哪？** 停用可逆、shadow entry 保留；刪除是帳號連同 entry 一起消失。
- **Oem 等級是什麼？** 保留給 OEM 專屬 command 的等級，比 Administrator 高；上游一般 command 不會用到。
- **為什麼 Redfish 的 role 跟 IPMI 的 privilege 名字不一樣？** 只是命名不同，底層都是 user-manager 同一個 privilege enum，`bmcweb` 做映射。
- **BMC 上有「group」概念嗎？** 上游沒有 group；授權一律以「單一個號的 privilege level」為準。
