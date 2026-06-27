---
title: Election1 Walkthrough 簡介
date: 2026-06-27
tags:
  - walkthrough
  - pg-play
  - linux
status: completed
difficulty: easy
---

# Election1 Walkthrough 簡介

本筆記摘要自詳細報告 [[Election1]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：開啟連接埠 `22 (SSH)` 與 `80 (HTTP)`。
*   **目錄爆破**：對 HTTP 服務爆破，在 `/robots.txt` 中發現 `/election` 目錄。
*   **二次目錄爆破**：對 `/election/` 進行爆破，尋獲 `/election/card.php` 與 `/election/admin/` 後台。

## 2. 初始存取 (Initial Access)
*   **憑證取得（第一階段）**：存取 `card.php`，將其顯示的二進位資料解碼兩次後，取得管理員帳密：`user:1234 / pass:Zxc123!@#`。
*   **憑證取得（第二階段）**：登入 `/election/admin/` 後台，於 `Settings > System Info > Logging` 中讀取 `system.log`，發現洩漏的 SSH 憑證：`love:P@$$w0rd@123`。
*   **Shell 取得**：使用 `ssh love@192.168.x.x` 成功登入系統並取得 `local.txt`。

## 3. 特權提升 (Privilege Escalation)
*   **SUID 提權**：執行 `linpeas` 發現 `/usr/local/Serv-U` 檔案具有 SUID 權限且存在已知漏洞（Serv-U FTP Server < 15.1.7 - CVE-2019-12181）。
*   **Exploit**：下載 `47173.sh` 本地提權腳本並執行，成功產出具有 root 權限的 `/tmp/sh`，進而獲取 `root` 權限與 `proof.txt`（亦可直接使用 `PwnKit` 提權）。
