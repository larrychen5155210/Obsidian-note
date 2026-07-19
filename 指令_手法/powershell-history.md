# 💥 PowerShell 歷史命令紀錄敏感資訊洩漏 (powershell-history)

本文件說明 Windows 環境中，如何藉由列舉與讀取 PowerShell (PSReadLine) 歷史紀錄檔來竊取敏感憑證或系統配置資訊。

---

## 🔍 漏洞成因與原理

在 Windows 系統中，PowerShell 預設使用 **`PSReadLine`** 模組來增強命令列的互動體驗（如自動補全與歷史查詢）。該模組會自動將使用者在終端機中輸入的每一條命令，以明文形式寫入硬碟中的特定檔案（`ConsoleHost_history.txt`）。

如果系統管理員、域管理員或其他高權限用戶在先前的操作中，於命令列中直接輸入了敏感資訊（例如：在執行指令或啟動服務時以參數傳遞密碼、使用 `runas`、或者是執行了帶有憑證的腳本），這些明文資訊就會被永久記錄在該檔案中。

當攻擊者取得某個用戶的初始 Shell 後，即使是低權限，只要能夠讀取該用戶（或管理員）的 `ConsoleHost_history.txt` 檔案，就能直接竊取其中洩漏的明文憑證，從而實現特權提升或橫向移動。

---

## 🛠️ 常見利用與繞過技巧

### 1. 獲取歷史紀錄檔路徑
在 PowerShell 終端機中，可以執行以下指令來直接查詢當前系統中歷史紀錄檔案的實際儲存絕對路徑：
```powershell
(Get-PSReadLineOption).HistorySavePath
```
*預設路徑通常為：*
`C:\Users\<USERNAME>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`

---

### 2. 檢視與列取歷史指令
使用 PowerShell 或 cmd 指令讀取該檔案的內容：
```powershell
# 使用 PowerShell 讀取
type "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"

# 或者是指定絕對路徑（例如讀取 Administrator 的歷史紀錄，如果當前權限允許）
type "C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[OSCP-C]]（在域成員主機 MS01 (192.168.x.153) 上，攻擊者取得本機管理員權限後，讀取了管理員的 PowerShell 歷史紀錄檔案 `ConsoleHost_history.txt`。在歷史命令中發現管理員曾執行過 `C:\users\support\admintool.exe hghgib6vHT3bVWf cmd`，其中參數帶有另一組管理員明文密碼：**`hghgib6vHT3bVWf`**。攻擊者隨後利用此憑證成功登入了內網成員主機 MS02 (10.10.x.154)）。
