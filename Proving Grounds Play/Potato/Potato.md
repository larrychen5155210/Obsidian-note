---
title: "Potato 簡介"
date: "2026-07-06"
tags:
  - pg-play
  - linux
  - ftp-anonymous
  - php-type-juggling
  - path-traversal
  - hash-cracking
  - sudo-exploitation
status: completed
difficulty: easy
---

# Potato 簡介

本筆記摘要自詳細報告 [[Potato-Walkthrough]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：確認開放連接埠 `22 (SSH)`、`80 (HTTP)` 與 `2112 (FTP)`。
*   **FTP 服務列舉**：偵測到 `2112/tcp` 允許匿名登入（Anonymous FTP login），下載並發現了備份檔案 `index.php.bak`。
*   **Web 目錄列舉**：爆破目錄發現敏感目錄 `/admin`，該目錄下有管理員登入頁面。

## 2. 初始存取 (Initial Access)
*   **漏洞利用 / 憑證獲取**：
    *   分析 `index.php.bak` 原始碼，發現登入驗證採用 `strcmp($_POST['password'], $pass) == 0`，存在 PHP 弱型別缺陷。
    *   利用 [[php-type-juggling#💥 PHP 弱類型比較與 strcmp 繞過 (php-type-juggling)|strcmp 弱類型比較繞過]]，將密碼參數修改為陣列 `password[]=` 傳遞，繞過驗證成功登入後台。
    *   在後台 `dashboard.php` 中發現日誌讀取功能存在漏洞，利用 [[path-traversal#💥 路徑/目錄走訪 (path-traversal)|目錄走訪]] 漏洞讀取 `/etc/passwd`，獲取了使用者 `webadmin` 的密碼 Hash（`$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/`）。
    *   使用 `john` 或 `hashcat` 工具對該 Hash 發起 [[hash-cracking#6. Linux md5crypt 密碼破解 (Mode 500)|md5crypt 離線破解]]，成功解密明文密碼為 `dragon`。
*   **Shell 取得**：使用憑證 `webadmin:dragon` 透過 SSH 登入，取得初始系統存取權（`local.txt`）。

## 3. 特權提升 (Privilege Escalation)
*   **權限提升向量**：
    *   執行 `sudo -l` 發現 `webadmin` 可以以 root 權限執行 `/bin/nice /notes/*`（需要密碼）。
    *   由於 `*` 通配符與 `nice` 指令本身均未限制相對路徑，利用 [[sudo#💥 Sudo 萬用字元與路徑走訪繞過 (Sudo Wildcard Path Traversal)|Sudo 萬用字元與路徑走訪繞過]] 手法，使用相對路徑走訪 `..` 逃逸 `/notes/` 目錄。
*   **取得 Root**：執行 `sudo /bin/nice /notes/../bin/bash`（或執行家目錄中寫有 `/bin/bash` 的自定義腳本），成功取得系統最高權限（`proof.txt`）。

## 🔗 資料來源 (References)
* [Potato Walkthrough - OffSec Proving Grounds Play](https://medium.com/@optimusprime7/potato-walkthrough-offsec-proving-grounds-play-87817f843484)
* [Potato Walkthrough - Proving Grounds](https://medium.com/@rizzziom/potato-walkthrough-proving-grounds-317688e9e811)
* [OffSec Proving Grounds: Potato](https://infosecwriteups.com/offsec-proving-grounds-potato-b080d38b4bed)
* [Potato Proving Grounds - OSCP Preparation](https://medium.com/@SilentExploit/potato-proving-grounds-oscp-preparation-57c2cfcdef5e)
* [OffSec Proving Grounds - Potato Writeup](https://hackmd.io/@lyws7/r1P1RjmGfe)
* [OffSec Proving Grounds - Potato Video Walkthrough](https://www.youtube.com/watch?v=9vEYhbtZmZY)
* [My Road to OSCP - Proving Grounds Play: Potato](https://medium.com/@sandrofrallicciardi/my-road-to-oscp-proving-grounds-play-warm-up-potato-164cd60354ef)
* [Potato - Proving Grounds Writeup](https://ziomsec.com/writeups/misc/potato/)
* [Potato (Easy) Linux Box Writeup](https://github.com/Bsal13/Offensive-Security-Proving-Grounds-Boxes/blob/main/Potato%20(Easy)%20Linux%20Box.md)
