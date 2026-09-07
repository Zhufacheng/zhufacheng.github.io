# phosphor-pid-control：重要 function 怎麼跑

這是 OpenBMC 上控制風扇轉速的 daemon：讀溫度 → 算出該給風扇的 PWM → 寫回風扇。

## 流程圖

```
main() 起程式
  │
  ├─ worker()            每 5s：讀 PID 參數檔 → ipmitool 寫進韌體
  │
  └─ restartControlLoops()   讀設定、建 sensor 和 zone、起迴路
         │
         ▼
  pidControlLoop()        每 200ms 醒一次
         │  讀風扇 tach
         │  每累積 1000ms 跑一次下面：
         ▼
  processThermals()
     ├─ updateSensors()            讀所有溫度
     ├─ processThermals(zone)      每顆溫度各跑一次 PID
     │     └─ PIDController::process()
     │          ├─ calPIDOutput() → ec::pid()   算出該 sensor 的 PWM
     │          └─ outputProc() → addSetPoint()
     ├─ determineMaxSetPointRequest()   取所有 sensor 的最大值
     └─ applyThermalOutputToFans()      決定最終 pwmDuty
            │
            ▼
     FanController::outputProc()    fail-safe 墊底、轉成 0~1
            │
            ▼
     DbusWritePercent::write()      寫 D-Bus  FanPwm.Target（0~100）
            │
            ▼
        風扇照這個轉速轉
```

## 重要 function（依執行順序）

### 起程式

- **`main()`**：讀命令參數（設定檔、log、debug 等）、建 D-Bus object manager、抓 `Hwmon.external` 和 `State.FanCtrl` 兩個 bus name、註冊 `signalHandler`、起 `worker()` thread、呼叫 `tryRestartControlLoops()` 起迴路、進 `io.run()`。
- **`worker()`**：獨立 thread，每 5 秒看一次 `/tmp/pid/pidParaOem.json`；有就讀一組組 PID 參數（Kp/Ki/Kd、上下限、setpoint、slew、up_inc/dw_inc、weight），用 `ipmitool raw 0x38` 寫進韌體，完事把檔案改名備份。這是「不重刷韌體就能換 PID 參數」的另一條路。
- **`restartControlLoops()`**：整套重啟。先停舊迴路、清顯示表；讀設定（本機 `config.json` 或 D-Bus/entity-manager）；`buildSensors` 建感測器、`buildZones` 建 zone；建 `SignalMonitor` 盯 threshold 和 PWM 變化；對每個 zone 建 timer、起 `pidControlLoop`。
- **`signalHandler()`**：SIGTERM 停機、SIGHUP 重讀設定重啟。

### 迴路

- **`pidControlLoop()`**：排程主體。用 timer 每 200ms 醒一次，每圈讀風扇 tach；累積到 1000ms 才跑一次 thermal（`processThermals` → `applyThermalOutputToFans`）；手動模式就跳過。遞迴排下一圈。
- **`processThermals()`**（pidloop）：每圈 thermal 依序：`updateSensors`（讀溫度）→ 跑每個 thermal controller → `determineMaxSetPointRequest`（定 max）。

### 每個 sensor 的 PID

- **`DbusPidZone::updateSensors()`**：讀所有溫度 sensor 進 cache。
- **`DbusPidZone::processThermals()`**：對 zone 內每個 thermal controller 跑 `process()`，把每顆的 output 記進顯示表。
- **`PIDController::process()`**：一個 sensor 的完整處理。`setptProc()` 拿目標、`inputProc()` 拿溫度、`calPIDOutput()` 算 output，再乘該 zone 的 weight、鎖到下限、`outputProc()` 送出。
- **`PIDController::calPIDOutput()`**：先查對應硬體有沒有裝（沒裝就跳過、不算），再走 hysteresis 判斷（高於目標一段才算、低於一段就歸零、帶內保持原值），呼叫 `ec::pid()`。
- **`ec::pid()`**：真正算 PWM。依 sensor 名讀 `/tmp/pid/pid_param_oem/<name>` 覆寫參數（沒有就用設定檔值，NVME/GPU 會依硬體換 setpoint）；算死區（帶內 error=0）；增量式 `P + I + D`；`output = 上次 + P + I + D`；clamp 到上下限；限速率（每圈最多 ±up_inc/±dw_inc）；依 zone 選 weight。
- **`ThermalController::inputProc()`**：把該 controller 的多個輸入聚合成一個值（margin 取 min、temp 取 max、power 求和）。
- **`ThermalController::outputProc(value)`**：`addSetPoint(value, 名字)`，送進 zone 去取 max。

### 取 max、送風扇

- **`DbusPidZone::addSetPoint()` / `determineMaxSetPointRequest()`**：每顆 sensor 各送一個「要幾%」，zone 取最大的（同組風扇只能一個轉速）；再跟 `minThermalOutput` 比取高，保證最低轉速。
- **`DbusPidZone::applyThermalOutputToFans()`**：`pwmDuty = 所有 sensor 的最大值`；若某些 fail 條件成立（風扇 offset、感測器沒讀數、超 threshold）就強制 100%；有 `/tmp/pid/manual` 可手動指定；最後 `fan->outputProc(pwmDuty)`。
- **`FanController::outputProc()`**：真正寫風扇。在 fail-safe 就把百分比墊到 fail-safe 值；`/100` 轉成 0~1；對每顆風扇 sensor `write()`。
- **`DbusWritePercent::write()`**：把 0~1 換回 0~100，寫 D-Bus `xyz.openbmc_project.Control.FanPwm.Target`（路徑 `/xyz/openbmc_project/control/fanpwm/Pwm_N`）。風扇照這個值轉。

### fail-safe

- **`DbusPidZone::initializeCache()` / `markSensorMissing()` / `getFailSafeMode()`**：開機時風扇當 fail-safe；sensor 讀不到或 timeout 就記為 missing；只要有任何 missing 或強制 fail-safe，`getFailSafeMode()` 就 true，`FanController::outputProc` 會把轉速墊高。

## 幾個重點

- **fan PID 階段不跑**：原本「thermal→目標轉速→fan PID 對 tach 再調」的第二段被砍掉，改由 `applyThermalOutputToFans` 直接把 thermal 的 max 當 PWM 送風扇。
- **寫風扇只走 D-Bus**：`FanController::outputProc` → `DbusWritePercent::write` → `FanPwm.Target`；code 裡那段用 shared memory 讀 tach 的邏輯是註解掉的。
- **讀不到 sensor 就 fail-safe**，風扇轉高，避免過熱。
- 很多行為看一堆 marker 檔切：power on/off（觸發重啟）、硬體 presence（沒裝就跳過）、fail 條件（強制 100%）、`/tmp/pid/manual`（手動）、`/tmp/pid/*`（PID 參數熱改、log 開關）。
