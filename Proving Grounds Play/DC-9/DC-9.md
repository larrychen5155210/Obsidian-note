---
title: "DC-9 簡介"
date: "2026-07-06"
tags:
  - vulnhub
  - pg-play
  - linux
  - sqli
  - path-traversal
  - port-knocking
  - ssh-bruteforce
  - sudo-exploitation
  - writable-etc-passwd
  - fuzzing
status: completed
difficulty: easy
---

# DC-9 簡介

本筆記摘要自詳細報告 [[DC-9-Walkthrough]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：確認開放連接埠 `80 (HTTP)`。
    *   *注意*：`22 (SSH)` 處於 `Filtered` 狀態，需通過連接埠敲擊觸發開啟。
*   **網頁功能分析**：在搜尋功能處發現 `search` 參數存在 SQL 注入點。
*   **目錄列舉**：爆破發現敏感頁面 `/manage.php`，該頁面在登入後存在 LFI（本地檔案包含）漏洞。

## 2. 初始存取 (Initial Access)
*   **資料庫列舉與後台登入**：
    *   利用 [[sqli#💥 SQL 注入 (SQLi)|Union-based SQLi]]，列舉 `Staff.Users` 資料表取得 `admin` 的 Hash 並破解得到密碼 `transorbital1`。
    *   使用 `admin` 憑證登入 `/manage.php` 網頁後台。
*   **檔案讀取與開啟 SSH (LFI & Port Knocking)**：
    *   使用 [[ffuf#💥 Web Fuzzing / 參數爆破 (fuzzing)|fuzzing]] 工具爆破出 `file` 參數，並在已登入的狀態下利用 [[path-traversal#💥 路徑/目錄走訪 (path-traversal)|路徑走訪]] 漏洞讀取 `/etc/knockd.conf`，取得 SSH 連接埠敲擊順序為 `7469, 8475, 9842`。
    *   利用 [[port-knocking#💥 連接埠敲擊 (port-knocking)|連接埠敲擊]] 機制，成功將 Filtered 的 SSH 22 埠開啟。
*   **憑證噴灑與登入**：
    *   再次透過 SQL 注入導出 `users.UserDetails` 資料表，取得多個內部使用者帳號與純文字密碼。
    *   以該帳密清單作為字典，利用 [[ssh-bruteforce#💥 SSH 服務密碼爆破 (ssh-bruteforce)|SSH 憑證噴灑]]，成功取得使用者 `janitor` 的 SSH 存取權。
*   **橫向移動 (janitor -> fredf)**：
    *   登入 `janitor` 後，在其家目錄發現隱藏的密碼檔，獲取額外的帳密。
    *   再度進行噴灑，取得使用者 `fredf:B4-Tru3-001` 的 SSH 權限並登入。

## 3. 特權提升 (Privilege Escalation)
*   **客製二進制漏洞**：
    *   執行 `sudo -l` 發現 `fredf` 擁有以 root 權限執行 `/opt/devstuff/dist/test/test` 的特權。
    *   審查發現該二進制檔會呼叫 `/opt/devstuff/test.py`，該腳本可將來源檔案內容「追加寫入」至目標檔案。
*   **取得 Root**：
    *   利用此任意檔案追加權限，套用 [[writable-etc-passwd#💥 可寫系統密碼檔提權 (writable-etc-passwd)|向 /etc/passwd 追加特權使用者]] 變體技巧。
    *   建立寫有 `pwned` 特權使用者（UID/GID 為 0）的臨時檔，將其寫入到 `/etc/passwd`，切換使用者 `su pwned` 成功獲取 root 權限與 `proof.txt`。

## 🔗 資料來源 (References)
* [Vulnhub | DC-9 Video Walkthrough](https://www.youtube.com/watch?v=x-8WT5vD6bg)
* [DC-9 Walkthrough](https://rafaelmedeiros94.medium.com/dc9-walkthrough-b3d1c4306)
* [Vulnhub | DC-9 Walkthrough](https://benheater.com/vulnhub-dc9/)
* [Vulnhub - DC-9 Writeup](https://publish.obsidian.md/d4rkc0de/writeup-all/vulnhub/01-linux/vulnhub-22-dc-9)
* [DC-9 Walkthrough](https://medium.com/@muhammedmidlaj430/dc-9-walkthrough-vulnhub-0d90e726b607)
* [VulnHub - DC-9 Video Guide](https://www.youtube.com/watch?v=fUfj5LXIHW0)
