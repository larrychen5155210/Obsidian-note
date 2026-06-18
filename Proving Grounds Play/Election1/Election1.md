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

##### 從根目錄開始爆破，找到 `phpinfo.php`、`robots.txt`

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

##### 在 `robots.tst` 裡找到一個其中一個目錄 `/election`，從這個目錄繼續嘗試爆破，且因為有找到 `phpinfo.php`，所以指定副檔名 `.php`，`-d 2` 限制只爆破到目錄第 2 層

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

