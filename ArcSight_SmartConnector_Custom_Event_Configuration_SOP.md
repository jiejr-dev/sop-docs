# ArcSight SmartConnector 自訂事件設定 SOP

## 文件目的
本文件用於指引維運人員在 Windows 主機上，為 ArcSight SmartConnector 設定自訂事件來源（Custom Event Logs），並完成設定檔移轉與服務重啟。

## 適用範圍
- ArcSight SmartConnector（Windows 環境）
- 需新增或調整自訂事件來源之主機

## 前置條件
- 以系統管理員權限登入目標主機
- 已安裝 ArcSight SmartConnector
- 可存取來源主機 `SDRSTP01` 的設定檔資料夾

## 作業步驟

### 1. 開啟設定介面
以系統管理員身分開啟 PowerShell，依實際安裝路徑執行以下任一指令：

```powershell
& "C:\Program Files\ArcSightSmartConnectors\windows\current\bin\runagentsetup.bat" -i swing
& "C:\Program Files\ArcSightSmartConnectors\chts\current\bin\runagentsetup.bat" -i swing
& "C:\Program Files\ArcSightSmartConnectors\ACSI\current\bin\runagentsetup.bat" -i swing
```

### 2. 修改 Connector 設定
在 Connector Setup 視窗中依序操作：

1. 選擇 `Modify Connector`，點選 `Next`
2. 選擇 `Modify Connector Parameters`，點選 `Next`
3. 勾選 `Custom Logs`，點選 `Next`
4. 在 `Custom Event Logs` 欄位輸入以下內容：

```text
Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational
```

5. 持續點選 `Next` 直到完成並關閉設定精靈

### 3. 複製設定檔
將來源主機 `SDRSTP01` 的下列資料夾內容複製到當前設備對應路徑：

```text
C:\Program Files\ArcSightSmartConnectors\chts\current\user\agent\fcp\winc
```

建議執行原則：
- 覆蓋前先備份本機原資料夾
- 確認檔案權限與擁有者設定未異常

### 4. 重啟服務
設定完成後，重啟 SmartConnector 相關服務。

服務名稱可能為以下其一：
- `ArcSight Microsoft Windows Event Log - Native`
- `ArcSight chts`

可於系統管理員 PowerShell 執行：

```powershell
# 查詢可能服務
Get-Service | Where-Object { $_.Name -like "*ArcSight*" -or $_.DisplayName -like "*ArcSight*" }

# 重啟候選服務（存在才會執行）
$serviceNames = @(
    "ArcSight Microsoft Windows Event Log - Native",
    "ArcSight chts"
)

foreach ($svc in $serviceNames) {
    if (Get-Service -Name $svc -ErrorAction SilentlyContinue) {
        Restart-Service -Name $svc -Force
    }
}
```

## 驗證項目
- Connector Setup 已完成且無錯誤訊息
- `Custom Event Logs` 已包含：
  - `Microsoft-Windows-TerminalServices-RemoteConnectionManager/Operational`
- 服務已成功重啟且狀態為 `Running`

可使用以下指令檢查服務狀態：

```powershell
Get-Service -Name "ArcSight Microsoft Windows Event Log - Native","ArcSight chts" -ErrorAction SilentlyContinue
```

## 例外處理
- 若找不到 `runagentsetup.bat`，請先確認實際安裝路徑與 Connector 版本
- 若服務名稱不同，請以 `Get-Service` 查詢結果為準後再重啟
- 若資料夾複製失敗，請確認防毒軟體或權限管制是否阻擋
