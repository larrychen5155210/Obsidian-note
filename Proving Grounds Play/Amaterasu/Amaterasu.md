---
title: Amaterasu 簡介
date: 2026-07-04
tags:
  - pg-play
  - linux
  - rest-api
  - file-upload
  - path-traversal
  - cron-job
  - path-hijacking
  - tar-wildcard
status: completed
difficulty: easy
---

# Amaterasu 簡介

本筆記摘要自詳細滲透測試報告 [[Amaterasu-Walkthrough]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：開放 FTP (21)、SSH (25022)、REST API (33414) 以及 HTTP (40080)。
*   **API 探測**：
    *   在 Port 33414 發現 Python Werkzeug 搭建的 REST API。
    *   目錄爆破發現 `/help` 檔案，洩漏以下 API 接口：
        *   `GET /info`
        *   `GET /help`
        *   `GET /file-list?dir=/tmp`
        *   `POST /file-upload`

## 2. 初始存取 (Initial Access)
*   **[[file-upload#💥 REST API 檔案上傳繞過與路徑穿越|REST API]] 檔案上傳與路徑穿越**：
    *   `/file-upload` 對 `file` 的副檔名設有白名單（僅允許 `txt, pdf, png, jpg, jpeg, gif` 等），但未過濾 `filename` 參數，導致存在 [[path-traversal#💥 路徑/目錄走訪 (path-traversal)|路徑穿越 (Path Traversal)]] 漏洞。
    *   將 Kali 的 SSH 公鑰命名為 `authorized_keys.txt` 並透過上傳繞過限制。
    *   將 `filename` 參數設為 `../home/alfredo/.ssh/authorized_keys`，直接覆寫用戶 `alfredo` 的 SSH 授權金鑰。
    *   利用私鑰成功以 `alfredo` 用戶身分登入 SSH (Port 25022)，並取得 `local.txt`。

## 3. 特權提升 (Privilege Escalation)
*   **[[cron-job#💥 定時任務腳本中的設計漏洞利用 (以 Amaterasu 為例)|Cron Job]] 與 Tar 參數注入/環境變數劫持**：
    *   使用 `pspy` 監控或檢查定時任務，發現 root 每分鐘會以相對路徑執行腳本 `/usr/local/bin/backup-flask.sh`。
    *   該腳本會先將 `/home/alfredo/restapi` 加進 `PATH` 最前端，並在該目錄下執行 `tar czf /tmp/flask.tar.gz *`。
    *   此處有兩種提權路徑：
        1.  **PATH 環境變數劫持**：在該目錄下建立惡意 `tar` 腳本，透過 [[path-hijacking#💥 PATH 環境變數劫持|PATH 劫持]] 執行任意指令。
        2.  **Tar 萬用字元參數注入**：在該目錄下建立 `--checkpoint=1` 與 `--checkpoint-action=exec=sh x` 檔案，利用 [[tar-wildcard#💥 Tar 萬用字元參數注入|Tar 萬用字元注入]] 執行惡意腳本，賦予 `/bin/bash` SUID 權限。
    *   執行 `bash -p` 成功提權至 `root` 取得 `proof.txt`。
