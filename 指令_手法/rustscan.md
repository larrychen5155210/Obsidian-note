# [安裝](https://github.com/bee-san/RustScan)
```bash
┌──(kali㉿kali)-[~/Downloads]
└─$ ls
x86_64-linux-rustscan.tar.gz.zip
```
```bash
unzip x86_64-linux-rustscan.tar.gz.zip 
```
```bash
tar -zxvf x86_64-linux-rustscan.tar.gz
```
```bash
sudo mv ./rustscan /usr/local/bin/
```
# 用法
```bash
rustscan -a 192.168.133.148 -- {nmap_參數}
```
```bash
rustscan -a 192.168.133.148 -- -sv
```
