**Windows Secure Boot 2023 憑證手動更新與維運指南 (Nutanix AHV / Hyper-V 環境適用)**  
**一、 前置檢查與狀態判定**  
在執行更新程序前，需先確認客體作業系統（Guest OS）內的安全開機已啟用。  
**1. 檢查安全開機（Secure Boot）啟用狀態**  
於管理員權限的 PowerShell 執行以下指令：  

```powershell
Confirm-SecureBootUEFI
```

- **預期結果**：回傳 `True`（代表功能已啟用）。  

- **異常排除（錯誤碼 0xC0000100 / STATUS_VARIABLE_NOT_FOUND）**：若出現此錯誤，代表虛擬韌體目前缺失 KEK 2023 金鑰，使 vUEFI 處於 **Setup Mode（設定模式）**。此為虛擬化平台常見的過渡現象，不影響核心 `db` 憑證之寫入。  

**2. 確認更新排程工作**  
檢查系統內建的憑證刷入排程是否就緒：  

```powershell
Get-ScheduledTask -TaskName "Secure-Boot-Update" -TaskPath "\Microsoft\Windows\PI\"
```

**二、 手動更新執行步驟**  
**步驟 1：配置登錄檔並觸發更新排程**  
透過登錄檔指定 2023 世代金鑰之更新計數（`0x5944`，即十進位 `22852`），並立即啟動背景更新工作：  

```powershell
# 1. 注入可用更新計數reg add HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Secureboot /v AvailableUpdates /t REG_DWORD /d 0x5944 /f# 2. 強制啟動憑證刷入排程Start-ScheduledTask -TaskName "\Microsoft\Windows\PI\Secure-Boot-Update"
```

**步驟 2：確認狀態變更**  
執行後靜置數分鐘，檢查數值與狀態變化：  

```powershell
# 檢查意欲更新之計數Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecureBoot" -Name AvailableUpdates# 檢查憑證暫存狀態Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecureBoot\Servicing" | Select UEFICA2023Status,UEFICA2023Error
```

- **預期進展**：`AvailableUpdates` 數值下降，且 `UEFICA2023Status` 由 `NotStarted` 變更為 `InProgress`。  

**步驟 3：多階段重啟與更新推進**  

1. **執行狀態與錯誤碼判定**：  
   • **情境 A（可執行冷重啟）**：當 `UEFICA2023Status` 顯示為 **InProgress** 且 `UEFICA2023Error` 為 **2147942750**（Error 350：需要重新啟動）時，代表該階段更新已暫停並進入掛起狀態，**即可執行冷重啟**。  
   • **情境 B（寫入受阻）**：若 `UEFICA2023Error` 顯示為 **2147942419**（Error 19：防寫保護），**可確認此作業系統尚未安裝 4 月份（或後續）累積安全性更新（KB）**，導致 Windows 無法順利寫入虛擬晶片。請先中止步驟並補強系統修補程式。  

2. **執行「冷開機」與範本重置流程**：  
   • *注意：於客體作業系統內點選「重新啟動」（熱開機）將無法完成虛擬晶片變數重載。*  
   • **Nutanix AHV 環境作法**：於 Windows 內關機 ➔ 確認 Nutanix Prism 管理介面 VM 狀態轉為 `Powered Off` ➔ 於 Prism 後台點擊 **Power On（外開機）**。  
   • **Hyper-V 環境作法（範本重置）**：於 Windows 內關機 ➔ 開啟 Hyper-V 管理員進入 VM 設定 ➔ 「安全 (Security)」分頁取消勾選「啟用安全開機」並套用 ➔ 將「範本」下拉選單切換為其他項目並儲存 ➔ 重新勾選「啟用安全開機」並將範本明確切換回 **「Microsoft Windows」** 並儲存 ➔ 執行外開機（此動作可強制重置虛擬 NVRAM 空間）。  

3. **重新手動觸發**：重新進入系統桌面後，再次手動觸發排程以消化剩餘憑證：  

```powershell
Start-ScheduledTask -TaskName "\Microsoft\Windows\PI\Secure-Boot-Update"
```

**三、 更新結果驗證**  
更新程序完成後，執行狀態檢查：  

```powershell
Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\SecureBoot\Servicing" | Select UEFICA2023Status,UEFICA2023Error
```

**1. 標準成功指標**  

- `UEFICA2023Status` = **Updated**  

- `UEFICA2023Error` = **0**（或完全空白無錯誤）  

**2. 環境特有的安全掛起指標（實質安全）**  
若冷重啟後狀態依然顯示以下數值，在實質維運層面同樣判定為安全（無過期死機風險）：  

- `UEFICA2023Status` = **InProgress**  

- `UEFICA2023Error` = **2147942750**  

- **原因解析**：此狀態代表 Windows 已成功將核心 `Windows UEFI CA 2023`（效期至 2035 年）寫入暫存區（即隨後檢查之 `db = True`）。此現象主因是底層虛擬化平台版本限制使其卡在 Setup Mode，無法完成最終握手，但不影響開機安全性。於 Nutanix AHV 環境下請維持現狀，切勿盲目切換 Secure Boot 開關，以免觸發 PCI 重新列舉導致網卡斷網（Nutanix KB-11141）。  

**四、 憑證與金鑰寫入狀態查詢指令**  
於管理員權限的 PowerShell 執行以下指令，實質驗證憑證與金鑰是否已成功寫入底層 NVRAM 晶片：  
**1. 驗證數位簽章資料庫（db）是否包含 2023 開機憑證**  

```powershell
([System.Text.Encoding]::ASCII.GetString((Get-SecureBootUEFI db).bytes) -match 'Windows UEFI CA 2023')
```

- **結果 True**：代表核心 `Windows UEFI CA 2023` 憑證已確認存在於晶片中，已免除過期無法開機之風險。  

- **結果 False**：代表憑證尚未成功寫入。  

**2. 驗證金鑰交換金鑰庫（KEK）是否包含 2023 更新金鑰**  

```powershell
([System.Text.Encoding]::ASCII.GetString((Get-SecureBootUEFI KEK).bytes) -match 'Microsoft Corporation KEK 2K CA 2023')
```

- **結果 True**：代表用來更新下一代憑證的通道金鑰已就緒。  

- **結果 False**：代表當前平台底層版本尚未支援此金鑰寫入（屬已知平台限制，不影響前述 `db=True` 之安全性，更新狀態仍會維持 `InProgress` 與 `2147942750`）。
