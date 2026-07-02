---
title: "InsanityHosting 簡介"
date: "2026-06-29"
tags:
  - pg-play
  - linux
  - sqli
  - firefox-decrypt
status: completed
difficulty: hard
---

# InsanityHosting 簡介

本筆記摘要自詳細報告 [[InsanityHosting-Walkthrough]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：確認開放連接埠 `21 (FTP)`、`22 (SSH)` 與 `80 (HTTP)`。
*   **服務與目錄列舉**：使用目錄爆破工具發現 `/news/`、`/monitoring/` 與 `/webmail/` (SquirrelMail)。
*   **使用者枚舉**：在 `/news/` 的發布者名稱中發現網站管理員 `otis`。

## 2. 初始存取 (Initial Access)
*   **漏洞利用 / 憑證獲取**：
    *   利用弱密碼嘗試，成功獲取 credentials `otis:123456`，並順利登入後台面板與郵件系統。
    *   在 `/monitoring/` 面板的 `Add Server` 功能中，發現二階 [[sqli|SQL 注入 (SQLi)]] 漏洞。
    *   利用 SQL 注入搭配 `load_file('/etc/passwd')` 讀取本地密碼檔，確認 `otis` 的登入 Shell 被限制為 `/sbin/nologin`，並鎖定擁有 `/bin/bash` 權限的本地使用者 `elliot`。
    *   轉向查詢資料庫系統的 `mysql.user` 表，成功 Dump 出 `elliot` 的 MySQL 密碼 Hash，並使用 CrackStation 線上秒破，得到密碼為 `elliot123`。
*   **Shell 取得**：透過 SSH 使用憑證 `elliot:elliot123` 成功登入系統，取得初始互動式 Shell（`local.txt`）。

## 3. 特權提升 (Privilege Escalation)
*   **權限提升向量**：於 `elliot` 家目錄中列舉，發現存有火狐瀏覽器的敏感憑證檔案夾 (`/home/elliot/.mozilla/firefox/`)，其目錄內含 `logins.json` 與 `key4.db`。
*   **取得 Root**：
    *   將火狐設定檔打包並下載至攻擊機，使用 [[firefox-decrypt|firefox_decrypt]] 工具在本地進行解密，成功獲取 root 的明文密碼 `S8Y389KJqWpJuSwFqFZHwfZ3GnegUa`。
    *   使用該密碼以 SSH 連接登入（或藉由 `su root`）取得 root 權限（`proof.txt`）。
