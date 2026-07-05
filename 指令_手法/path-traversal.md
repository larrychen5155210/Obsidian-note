# 💥 路徑/目錄走訪 (path-traversal)

本文件說明目錄走訪 (Directory Traversal / Path Traversal) 漏洞的成因、檢測方法與常見的利用與繞過技巧。

---

## 🔍 漏洞成因與原理

當網頁應用程式允許使用者透過參數指定要讀取或下載的檔案名稱，而後端程式碼在讀取檔案前**未進行嚴格的輸入過濾、路徑限制或沙盒化**（如未使用 `basename()` 或是未限制檔案根目錄）時，攻擊者可以利用相對路徑走訪字元（如 `../` 或 `..\`）來逃逸目前的網頁根目錄，進而讀取或寫入伺服器上的敏感檔案。

常見的敏感檔案包含：
*   **Linux**：`/etc/passwd` (使用者清單)、`/etc/shadow` (密碼 Hash，若權限足夠)、`~/.ssh/id_rsa` (SSH 私鑰)、Web 設定檔等。
*   **Windows**：`C:\Windows\win.ini`、`C:\Windows\System32\drivers\etc\hosts` 等。

---

## 🛠️ 常見利用與繞過技巧

### 1. 基本利用
通常出現在參數如 `?file=`, `?page=`, `?path=`, `?doc=` 等：
```http
http://target.com/dashboard.php?file=../../../../../../etc/passwd
```

### 2. Burp Suite 爆破目錄深度
有時由於目錄結構較深，或是參數可能需要特定前綴，可以使用 Burp Suite 的 **Intruder** 工具，載入 Directory Traversal 的字典檔（例如 Seclists 中的 `raft-large-directories.txt` 或專門的 traversal 字典）進行自動化爆破。

### 3. 常見過濾繞過
*   **雙重 URL 編碼**：
    *   `../` 轉換為 `%252e%252e%252f` 或 `%252e%252e%255c`
*   **巢狀過濾繞過**（如果後端僅簡單地將 `../` 取代為空字串）：
    *   使用 `....//`（過濾後會變成 `../`）
    *   使用 `..././`
*   **空字元截斷**（適用於 PHP < 5.3.4，用以繞過後端強加的副檔名限制）：
    *   使用 `%00`（例如 `../../etc/passwd%00.jpg`）

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Potato]]（在管理後台 `dashboard.php` 中，利用 `file` 參數存在目錄走訪漏洞，透過 Burp Suite 爆破深度，成功讀取包含 `webadmin` 的密碼 Hash 的 `/etc/passwd` 檔案）。
    *   [[Amaterasu]]（在 Port 33414 的 REST API 檔案上傳端點中，未過濾 `filename` 參數，導致存在路徑穿越漏洞，可用於上傳 SSH 公鑰覆寫使用者 `alfredo` 的授權金鑰）。
