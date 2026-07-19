# 💥 密碼與雜湊值破解 (hash-cracking)

本文件彙整在滲透測試中，使用 `hashcat` 與 `john the ripper` 進行密碼與各種雜湊值 (Hash) 離線破解的指令與模式。

---

## 🔍 漏洞成因與原理

在滲透過程中，攻擊者常會透過各種方式（如資料庫洩漏、記憶體轉儲、登錄檔備份、網路攔截或協定漏洞）獲取加密的密碼雜湊或加密私鑰。為獲取明文憑證以進行橫向移動或權限提升，必須在本地使用高性能計算設備對這些雜湊進行字典爆破或規則破解。

*   **Hashcat**：基於 GPU 的高速雜湊破解工具，支援數百種雜湊算法，透過指定 `-m <Mode>` 來選擇對應的算法類型。
*   **John the Ripper (John)**：基於 CPU 的強大密碼破解工具，常配合各種轉換工具（如 `ssh2john`、`keepass2john`）將特殊格式的加密檔案轉換為 John 可識別的雜湊。

---

## 🛠️ 常見利用與繞過技巧

### 1. MD5 雜湊破解 (Mode 0)
常見於傳統資料庫（如 MySQL 舊版、SQLite 備份等）中的用戶密碼欄位。
*   **Hashcat 破解**：
    ```bash
    hashcat -m 0 <hash_file_or_hash> /usr/share/wordlists/rockyou.txt
    ```

### 2. Kerberos AS-REP 破解 (Mode 18200)
用於 AS-REP Roasting 攻擊，針對無須 Pre-authentication 的網域用戶所請求到的 Kerberos 5 AS-REP (etype 23) 雜湊。
*   **Hashcat 破解**：
    ```bash
    hashcat -m 18200 <hash_file> /usr/share/wordlists/rockyou.txt
    ```
*   **John 破解**：
    ```bash
    john --wordlist=/usr/share/wordlists/rockyou.txt <hash_file>
    ```

### 3. Kerberos TGS-REP 破解 (Mode 13100)
用於 Kerberoasting / Targeted Kerberoasting 攻擊，針對網域服務帳戶所請求到的 Kerberos 5 TGS-REP (etype 23) 雜湊。
*   **Hashcat 破解**：
    ```bash
    hashcat -m 13100 <hash_file> /usr/share/wordlists/rockyou.txt
    ```
*   **John 破解**：
    ```bash
    john --wordlist=/usr/share/wordlists/rockyou.txt <hash_file>
    ```

### 4. WordPress (MD5 / phpass) 破解 (Mode 400)
用於破解 WordPress 資料庫（如 `wp_users`）中以 `$P$` 或 `$H$` 開頭的加密密碼。
*   **Hashcat 破解**：
    ```bash
    hashcat -m 400 <hash_file> /usr/share/wordlists/rockyou.txt
    ```

### 5. SSH 加密私鑰 (Passphrase) 破解
當 SSH 私鑰被密碼保護時，可以使用 John 進行解密。
1.  **將私鑰轉換為 John 格式**：
    ```bash
    ssh2john id_rsa > id_rsa.hash
    ```
2.  **使用 John 進行字典爆破**：
    ```bash
    john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
    ```

### 6. Linux md5crypt 密碼破解 (Mode 500)
常見於 Linux 系統的 `/etc/passwd`（舊版中直接寫入 hash）或 `/etc/shadow` 中，以 `$1$` 開頭的 md5crypt 加密雜湊。
*   **Hashcat 破解**：
    ```bash
    hashcat -m 500 <hash_file> /usr/share/wordlists/rockyou.txt
    ```
*   **John 破解**：
    ```bash
    john --wordlist=/usr/share/wordlists/rockyou.txt <hash_file>
    ```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[OSCP-C]]（在 DMZ 滲透中，從 SQLite 資料庫中提取出 `support` 用戶的 MD5 雜湊，使用 `hashcat -m 0` 破解出明文密碼 `Freedom1`）。
    *   [[Poseidon]]（發起 AS-REP Roasting 取得帳戶 `chen` 的 hash，使用 `hashcat -m 18200` 破解出密碼 `freedom`）。
    *   [[Beyond]]（對網域服務帳戶發起 Kerberoasting，取得 TGS hash，使用 `hashcat -m 13100` 破解出密碼 `DANIelaRO123`；另外，讀取到受密碼保護的 SSH 私鑰，使用 `ssh2john` 與 `john` 破解出密碼 `tequieromucho` 成功登入）。
    *   [[Laser]]（發起 Targeted Kerberoasting 取得 `boris.crawford` 的 TGS hash，使用 `hashcat -m 13100` 成功破解出 `zxcvbnm`）。
    *   [[Blogger]]（從 WordPress 資料庫的 `wp_users` 欄位提取出 phpass 雜湊，使用 `hashcat -m 400` 破解出明文密碼，藉此登入後台取得 Web Shell）。
    *   [[Potato]]（透過目錄走訪漏洞讀取到 `/etc/passwd` 檔案，發現 `webadmin` 的 `$1$` 開頭 md5crypt 雜湊，使用 `john`（對應 `hashcat -m 500`）成功破解出明文密碼 `dragon`，隨後以 SSH 成功登入）。
