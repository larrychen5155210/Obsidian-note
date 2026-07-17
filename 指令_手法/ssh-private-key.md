# SSH 私鑰不安全權限利用 (ssh-private-key)
本文件說明如何利用系統中權限設定錯誤或外洩的 SSH 私鑰（`id_rsa`）進行橫向移動（Horizontal Movement）或初始存取（Initial Access）。

---

## 🔍 漏洞檢測與憑證發現
在已取得權限的 Shell 中，檢查常見的使用者 SSH 目錄是否存有可讀的私鑰檔案：
```bash
# 檢查特定使用者的 .ssh 目錄
ls -la /home/<username>/.ssh/

# 常見可讀私鑰路徑範例
cat /home/<username>/.ssh/id_rsa
cat /home/<username>/.ssh/id_dsa
```
若私鑰檔案對當前低權限使用者具有讀取權限（世界可讀或群組可讀），即可將其內容複製出來。

---

## 🛠️ 利用步驟

### 1. 複製並儲存私鑰
將目標系統上的私鑰內容複製，並在你的攻擊機上建立一個新檔案（例如 `id_rsa`），將私鑰內容完整貼入。

### 2. 設定安全權限
SSH 客戶端要求私鑰檔案必須具有嚴格的權限設定。若權限過於寬鬆（例如 `644` 或 `777`），SSH 連線將會被拒絕。
```bash
chmod 600 id_rsa
```

### 3. 進行 SSH 登入
使用 `-i` 參數指定該私鑰進行登入：
```bash
ssh -i id_rsa <username>@<target_ip>
```

### 4. 破解加密私鑰 (Passphrase Protected Key)
若私鑰設定了密碼保護 (Passphrase)，在使用 SSH 登入時會提示輸入密碼。此時可使用 `john` 工具進行本地破解：
1. **將私鑰轉換為 John 可識別的 Hash 格式**：
   ```bash
   ssh2john id_rsa > id_rsa.hash
   ```
2. **使用字典（如 rockyou.txt）進行破解**：
   ```bash
   john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
   ```
3. **使用破解出的密碼配合私鑰進行 SSH 登入**。

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[SoSimple]]（透過 `www-data` 權限發現 `/home/max/.ssh/id_rsa` 權限設定錯誤為世界可讀，下載該私鑰後成功 SSH 登入為 `max` 使用者）。
    *   [[Beyond]]（利用外掛漏洞讀取 `daniela` 的加密私鑰，使用 `ssh2john` 與 `john` 破解出其 Passphrase `tequieromucho` 後成功 SSH 登入）。

