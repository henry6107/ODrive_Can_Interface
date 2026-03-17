# ODrive_Can_Interface

這個專案是一個 TwinCAT 3 PLC 範例，將 ODrive 的 CAN Simple 指令封裝成可直接在 PLC 內呼叫的功能塊。主要使用入口是 `ODriveAxisController`，使用者只需要在週期任務中執行此功能塊，並呼叫對應方法，就可以完成 ODrive 軸狀態切換、控制模式設定、位置/速度命令下發，以及心跳、編碼器與錯誤資訊讀取。

目前版本另外支援 ODrive cyclic heartbeat 的被動接收與自動快取。當 ODrive 端已啟用 cyclic heartbeat 時，PLC 可以不主動發送 `GetHeartBeat` 請求，而是直接透過 `bEnableAutoHeartBeat` 監看最新 heartbeat 狀態。

目前專案內的通訊路徑如下：

- `ODriveAxisController`：對外提供控制與讀取方法。
- `ODriveCanSimpleTransporter`：負責把方法呼叫轉成 CAN Queue 收發。
- EL6751 CAN Queue 映射：透過 `%Q*` / `%I*` 與 PLC 結構體交換資料。
- 自動 heartbeat 快取：被動監看 cyclic heartbeat，更新 `stAutoHeartBeat` 與有效性旗標。

## 適用環境

- TwinCAT 3 專案
- Beckhoff EL6751 CAN 介面
- 已啟用 CAN Simple 的 ODrive
- PLC 專案內已包含本 repo 的 POU、DUT、GVL 與 I/O 設定

## 專案中的既有假設

本專案目前依賴以下設定，整合時請保持一致：

- CAN Queue 長度為 10 筆 Tx 與 10 筆 Rx
- PLC 結構 `ST_CanQueue` 對應 EL6751 的 Queue 結構
- `ODriveCanSimpleTransporter` 使用 `%Q*` 對應 Tx Queue、`%I*` 對應 Rx Queue
- 自動 heartbeat timeout 使用 `GVL_ODrive.nAutoHeartBeat_TimeOut`，目前為 `T#500MS`
- 全域 timeout 使用 `GVL_ODrive.nReceiveData_TimeOut`，目前為 `T#1S`
- 專案中已配置 PLC Task 與 EL6751 I/O

如果 EL6751 的 Queue 結構、資料長度或映射方式與 `ST_CanQueue` 不一致，PLC 雖然可以編譯，但命令收發會失敗。

## 快速開始

建議依下列順序確認整合環境：

1. 用 TwinCAT 開啟 solution：`ODrive_Can_Interface.sln`
2. 確認 PLC 專案與 I/O 設定都已成功載入
3. 確認 EL6751 已使用 CAN Queue 模式，且 Queue 長度與專案一致
4. 確認 EL6751 的 CAN 參數與 ODrive 端一致，例如 baud rate、node ID 與實際匯流排連線
5. 確認 PLC 程式中傳入的 `NodeId` 與 ODrive 設定一致
6. Activate Configuration、下載 PLC、切到 Run 模式

## 如何使用 `ODriveAxisController`

### 1. 宣告功能塊

最基本的使用方式如下：

```iecst
VAR
    _fbODriveAxis : ODriveAxisController;
    bEnableAutoHeartBeat : BOOL := TRUE;
END_VAR
```

此功能塊必須在每個 scan 週期都被執行，並傳入目標 ODrive 的 `NodeId`。若要被動接收 cyclic heartbeat，請同時開啟 `bEnableAutoHeartBeat`：

```iecst
_fbODriveAxis(
    NodeId := 7,
    bEnableAutoHeartBeat := bEnableAutoHeartBeat
);
```

如果沒有在循環內持續執行這一行，內部的 `ODriveCanSimpleTransporter` 不會持續推進狀態機，命令可能無法送出或完成。

### 2. 方法呼叫的共通模式

`ODriveAxisController` 的每個方法都採用相同的非阻塞狀態機介面：

- `bExecute`：上升沿觸發一次命令
- `bBusy`：命令處理中
- `bDone`：命令成功完成
- `bError`：命令失敗
- `sErrorMsg`：錯誤訊息

使用規則如下：

1. 將 `bExecute` 從 `FALSE` 拉成 `TRUE` 以觸發命令
2. 等待 `bDone = TRUE` 或 `bError = TRUE`
3. 命令完成後，將 `bExecute` 拉回 `FALSE`
4. 等方法回到 idle 後，才能再次觸發同一命令

如果 `bExecute` 一直維持 `TRUE`，方法會停留在完成狀態，無法重新用新的上升沿再觸發一次。

### 3. 自動 HeartBeat 更新

若 ODrive 已設定每 100 ms 送出 cyclic heartbeat，可直接使用 `ODriveAxisController` 的自動快取輸出：

- `stAutoHeartBeat`：最後一次收到的 heartbeat 資料
- `bAutoHeartBeatValid`：目前快取資料有效
- `bAutoHeartBeatUpdated`：本 scan 剛收到新的 heartbeat
- `bAutoHeartBeatTimedOut`：超過 `GVL_ODrive.nAutoHeartBeat_TimeOut` 沒收到新 heartbeat

這個功能是被動監聽，不會主動發送 CAN 請求。若 `bEnableAutoHeartBeat = FALSE`，這些狀態旗標會清為 `FALSE`，但保留最後一次收到的 heartbeat 內容。

### 4. 基本建議操作順序

第一次接上 ODrive 時，建議依照下列順序操作：

1. `ClearErrors`
2. `SetControllerMode`
3. `SetAxisState(ClosedLoopControl)`
4. `SetInputPos` 或 `SetInputVel`

這個順序與專案中的 sample 程式一致，比較符合一般軸控制流程。

## 使用範例

以下範例整理自 `ODRIVE_SAMPLE`，示範一個基本的位置控制流程。

### 宣告變數

```iecst
VAR
    _fbODriveAxis : ODriveAxisController;
    bEnableAutoHeartBeat : BOOL := TRUE;

    bClearErrorsExecute : BOOL;
    byClearErrorsIdentify : BYTE;
    bClearErrorsBusy : BOOL;
    bClearErrorsError : BOOL;
    sClearErrorsErrorMsg : STRING;
    bClearErrorsDone : BOOL;

    bSetControllerModeExecute : BOOL;
    eRequestedControlMode : E_OdriveControlMode := E_OdriveControlMode.Position;
    eRequestedInputMode : E_OdriveInputMode := E_OdriveInputMode.Passthrough;
    bSetControllerModeBusy : BOOL;
    bSetControllerModeError : BOOL;
    sSetControllerModeErrorMsg : STRING;
    bSetControllerModeDone : BOOL;

    bSetAxisStateExecute : BOOL;
    eRequestedAxisState : E_OdriveAxisState := E_OdriveAxisState.ClosedLoopControl;
    bSetAxisStateBusy : BOOL;
    bSetAxisStateError : BOOL;
    sSetAxisStateErrorMsg : STRING;
    bSetAxisStateDone : BOOL;

    bSetInputPosExecute : BOOL;
    rRequestedInputPos : REAL;
    nRequestedInputPosVelFF : INT;
    nRequestedInputPosTorqueFF : INT;
    bSetInputPosBusy : BOOL;
    bSetInputPosError : BOOL;
    sSetInputPosErrorMsg : STRING;
    bSetInputPosDone : BOOL;

    stAutoHeartBeat : ST_OdriveHeartbeat;
    bAutoHeartBeatValid : BOOL;
    bAutoHeartBeatUpdated : BOOL;
    bAutoHeartBeatTimedOut : BOOL;

    bReadStatusExecute : BOOL;
    stOdriveEncoderEstimates : ST_OdriveEncoderEstimates;
    stOdriveError : ST_OdriveError;
END_VAR
```

### 週期呼叫

```iecst
_fbODriveAxis.ClearErrors(
    bExecute := bClearErrorsExecute,
    byIdentify := byClearErrorsIdentify,
    bBusy => bClearErrorsBusy,
    bError => bClearErrorsError,
    sErrorMsg => sClearErrorsErrorMsg,
    bDone => bClearErrorsDone
);

_fbODriveAxis.SetControllerMode(
    bExecute := bSetControllerModeExecute,
    eControlMode := eRequestedControlMode,
    eInputMode := eRequestedInputMode,
    bBusy => bSetControllerModeBusy,
    bError => bSetControllerModeError,
    sErrorMsg => sSetControllerModeErrorMsg,
    bDone => bSetControllerModeDone
);

_fbODriveAxis.SetAxisState(
    bExecute := bSetAxisStateExecute,
    eRequestedState := eRequestedAxisState,
    bBusy => bSetAxisStateBusy,
    bError => bSetAxisStateError,
    sErrorMsg => sSetAxisStateErrorMsg,
    bDone => bSetAxisStateDone
);

_fbODriveAxis.SetInputPos(
    bExecute := bSetInputPosExecute,
    rInputPos := rRequestedInputPos,
    nVelFF := nRequestedInputPosVelFF,
    nTorqueFF := nRequestedInputPosTorqueFF,
    bBusy => bSetInputPosBusy,
    bError => bSetInputPosError,
    sErrorMsg => sSetInputPosErrorMsg,
    bDone => bSetInputPosDone
);

_fbODriveAxis.GetEncoderEstimates(
    bExecute := bReadStatusExecute,
    stOdriveEncoderEstimates => stOdriveEncoderEstimates
);

_fbODriveAxis.GetError(
    bExecute := bReadStatusExecute,
    stOdriveError => stOdriveError
);

_fbODriveAxis(
    NodeId := 7,
    bEnableAutoHeartBeat := bEnableAutoHeartBeat
);

stAutoHeartBeat := _fbODriveAxis.stAutoHeartBeat;
bAutoHeartBeatValid := _fbODriveAxis.bAutoHeartBeatValid;
bAutoHeartBeatUpdated := _fbODriveAxis.bAutoHeartBeatUpdated;
bAutoHeartBeatTimedOut := _fbODriveAxis.bAutoHeartBeatTimedOut;
```

若要週期性更新心跳狀態，建議使用 `bEnableAutoHeartBeat` 的被動接收。若要主動讀取 `GetEncoderEstimates` 或 `GetError`，仍然請讓 `bReadStatusExecute` 以脈衝方式觸發，而不是長時間維持 `TRUE`。

### 簡單的命令觸發觀念

以下示意如何對 `SetAxisState` 送出一次命令：

```iecst
IF bStartClosedLoop AND NOT bSetAxisStateBusy AND NOT bSetAxisStateDone THEN
    bSetAxisStateExecute := TRUE;
END_IF

IF bSetAxisStateDone OR bSetAxisStateError THEN
    bSetAxisStateExecute := FALSE;
END_IF
```

實務上你也可以用自己的步序控制，重點是要保留上升沿觸發與完成後復位的邏輯。

## 常用方法說明

### 功能塊輸入 / 輸出

| 名稱 | 類型 | 用途 |
| --- | --- | --- |
| `NodeId` | FB Input | 指定目標 ODrive CAN Node ID |
| `bEnableAutoHeartBeat` | FB Input | 啟用被動接收 cyclic heartbeat |
| `stAutoHeartBeat` | FB Output | 最後一次收到的 heartbeat 資料 |
| `bAutoHeartBeatValid` | FB Output | heartbeat 快取目前有效 |
| `bAutoHeartBeatUpdated` | FB Output | 本 scan 剛收到新 heartbeat |
| `bAutoHeartBeatTimedOut` | FB Output | heartbeat 超時未更新 |

### 控制類方法

| 方法 | 用途 | 主要輸入 | 主要輸出 |
| --- | --- | --- | --- |
| `ClearErrors` | 清除 ODrive 錯誤 | `bExecute`, `byIdentify` | `bBusy`, `bDone`, `bError`, `sErrorMsg` |
| `Estop` | 緊急停止 | `bExecute` | `bBusy`, `bDone`, `bError`, `sErrorMsg` |
| `SetAxisState` | 切換軸狀態 | `bExecute`, `eRequestedState` | `bBusy`, `bDone`, `bError`, `sErrorMsg` |
| `SetControllerMode` | 設定控制模式與輸入模式 | `bExecute`, `eControlMode`, `eInputMode` | `bBusy`, `bDone`, `bError`, `sErrorMsg` |
| `SetInputPos` | 下發位置命令 | `bExecute`, `rInputPos`, `nVelFF`, `nTorqueFF` | `bBusy`, `bDone`, `bError`, `sErrorMsg` |
| `SetInputVel` | 下發速度命令 | `bExecute`, `rInputVel`, `rInputTorqueFF` | `bBusy`, `bDone`, `bError`, `sErrorMsg` |
| `SetAbsolutePosition` | 設定絕對位置 | `bExecute`, `rPosition` | `bBusy`, `bDone`, `bError`, `sErrorMsg` |

### 讀取類方法

| 方法 | 用途 | 主要輸入 | 回傳資料 |
| --- | --- | --- | --- |
| `GetHeartBeat` | 讀取心跳與狀態 | `bExecute` | `ST_OdriveHeartbeat` |
| `GetEncoderEstimates` | 讀取位置/速度估測值 | `bExecute` | `ST_OdriveEncoderEstimates` |
| `GetError` | 讀取錯誤資訊 | `bExecute` | `ST_OdriveError` |

## 常用型別

以下型別在整合時最常用：

- `E_OdriveAxisState`
  - 例如：`Idle`、`ClosedLoopControl`
- `E_OdriveControlMode`
  - 例如：`Voltage`、`Torque`、`Velocity`、`Position`
- `E_OdriveInputMode`
  - 例如：`Passthrough`、`VelRamp`、`TrapTraj`
- `ST_OdriveHeartbeat`
  - 包含 `dwAxisError`、`eAxisState`、`eProcedureResult`、`byTrajectoryDone`
- `ST_OdriveEncoderEstimates`
  - 包含 `rPosEstimate`、`rVelEstimate`
- `ST_OdriveError`
  - 包含 `dwActiveErrors`、`dwDisarmReason`

## 整合注意事項

### `NodeId`

`ODriveAxisController(NodeId := ...)` 中的 `NodeId` 必須與 ODrive 實機設定一致，否則即使 CAN 匯流排正常，PLC 也會因為收不到對應節點的回覆而 timeout。

### Timeout

所有等待回應的方法都會受到 `GVL_ODrive.nReceiveData_TimeOut` 影響，目前預設值是 `T#1S`。若通訊正常但設備回覆較慢，可以視需要調整此常數。

自動 heartbeat 快取另外使用 `GVL_ODrive.nAutoHeartBeat_TimeOut`，目前預設值是 `T#500MS`。若你的 ODrive heartbeat 週期不是 100 ms，可以依實際情況調整這個 timeout。

### Queue 上限

`ODriveCanSimpleTransporter` 的傳送佇列上限是 10 筆。若同一時間塞入過多命令，`AddCmd` 會回報：

```text
Tx message queue is fulled !
```

因此不建議在同一個 scan 內對多個方法同時連續觸發大量命令。

### EL6751 映射

本專案依賴 EL6751 的 CAN Queue 結構與 PLC 內的 `ST_CanQueue` 一致。若你把這套程式搬到其他專案，請優先檢查：

- Tx Queue / Rx Queue 是否仍為 10 筆
- Queue 內每筆訊息是否仍為 `CobId + 8 bytes data`
- `%Q*` 與 `%I*` 是否仍正確映射到 EL6751 Queue 區域

## 疑難排解

### 1. 一直 timeout

請依序檢查：

- `NodeId` 是否與 ODrive 設定一致
- EL6751 是否已正常進入運作狀態
- ODrive 是否真的啟用 CAN Simple
- CAN baud rate 是否一致
- EL6751 Queue 映射是否與 `ST_CanQueue` 相容

### 2. 方法只成功一次，之後無法再次執行

通常是 `bExecute` 沒有在完成後拉回 `FALSE`。本專案的方法都依賴上升沿觸發，必須先回到 idle 才能再次啟動。

### 3. 自動 heartbeat 一直無效或 timeout

請檢查：

- ODrive 是否已啟用 cyclic heartbeat
- `bEnableAutoHeartBeat` 是否為 `TRUE`
- `NodeId` 是否正確
- `GVL_ODrive.nAutoHeartBeat_TimeOut` 是否太短
- EL6751 是否確實收到 heartbeat frame

### 4. 命令送不出去或 `bDone` 很慢才出現

請檢查：

- 是否有太多命令同時進入 Queue
- `_fbODriveAxis(NodeId := ...)` 是否真的每個 scan 都有執行
- PLC task 是否正常運作

### 5. 可以讀狀態，但動作命令沒有效果

請確認：

- 是否已先 `ClearErrors`
- 是否已先 `SetControllerMode`
- 是否已進入 `E_OdriveAxisState.ClosedLoopControl`
- 位置或速度命令的模式是否與當前控制模式相符

## 安全提醒

- 實際測試前請先確認機構、負載與人員安全
- 建議先確保 `Estop` 可正常使用，再開始測試位置或速度命令
- 第一次上機時，請先用低風險條件測試，不要直接在高速度或高扭矩下驗證

## 參考程式

若要看完整範例，請參考 PLC 專案中的 `ODRIVE_SAMPLE`：

- `ODrive_Can_Interface/Untitled1/POUs/ODRIVE_SAMPLE.TcPOU`

主要控制功能塊位於：

- `ODrive_Can_Interface/Untitled1/POUs/ODrive/ODriveAxisController.TcPOU`

底層 CAN Queue 傳輸功能塊位於：

- `ODrive_Can_Interface/Untitled1/POUs/ODrive/ODriveCanSimpleTransporter.TcPOU`

## 參考資料

- ODrive CAN Protocol Overview
  - https://docs.odriverobotics.com/v/latest/manual/can-protocol.html#overview
- Beckhoff CAN Interface 文件總覽
  - https://infosys.beckhoff.com/content/1033/can-interface/index.html?id=3488687489320045389
- Beckhoff EL6751 CAN queue / communication 說明
  - https://infosys.beckhoff.com/content/1033/el6751/2519175947.html?id=5544932967770383589
