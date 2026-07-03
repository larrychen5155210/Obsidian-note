---
title: "Gaara 簡介"
date: "2026-07-03"
tags:
  - pg-play
  - linux
  - ssh-bruteforce
  - suid
  - gdb
  - base58
status: completed
difficulty: easy
---

# Gaara 簡介

本筆記摘要自詳細報告 [[Gaara-Walkthrough]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：開放 TCP 連接埠 `22 (SSH)` 與 `80 (HTTP)`。
*   **目錄列舉**：使用預設的小字典（如 `common.txt`）無法掃描出有效結果。需改用較大的字典（例如 `directory-list-2.3-medium.txt`）進行目錄爆破，成功發現隱藏目錄 `/Cryoserver`。
*   **敏感資訊對比**：在 `/Cryoserver` 下發現三個頁面 `/Temari`、`/Kazekage` 與 `/iamGaara`。透過對比網頁內容，發現 `/iamGaara` 結尾多出了一串 [[base64-base58#🔍 Base58 編碼與解碼|Base58 編碼字串]]（解碼為一組憑證，但為 Rabbit Hole）。

## 2. 初始存取 (Initial Access)
*   **憑證獲取**：由於取得了可能的使用者名稱 `gaara`（同時也是靶機名稱），使用 [[ssh-bruteforce#🔍 Hydra SSH 密碼爆破|Hydra SSH 密碼爆破]] 手法針對 SSH 進行暴力破解。
*   **Shell 取得**：成功爆破出憑證 `gaara:iloveyou2`，藉此透過 SSH 登入系統並取得 `local.txt`。

## 3. 特權提升 (Privilege Escalation)
*   **權限提升向量**：執行本地枚舉，發現 `/usr/bin/gdb` 檔案具備不常見的 `SUID` 權限。
*   **取得 Root**：參考 [[suid#💥 gdb SUID 權限逃逸 (gdb)|gdb SUID 逃逸]] 手法，使用其內建的 Python 模組執行具備特權模式的 Shell，成功取得 `root` 權限並讀取 `proof.txt`。

## 🔗 資料來源 (References)
* [靶场实战(13)：OSCP备考之VulnHub GAARA - 腾讯云开发者社区](https://cloud.tencent.com/developer/article/2457825)
* [Gaara Walkthrough. OSCP Series | by thegeekaman - Medium](https://medium.com/@thegeekaman/gaara-walkthrough-oscp-series-9ce4dc7fee00)
* [Gaara Proving Grounds Play Walkthrough - YouTube](https://www.youtube.com/watch?v=zHYoZ4iocrU)
* [Offensive Security Proving Grounds: Gaara | by Software Sinner - Medium](https://software-sinner.medium.com/offensive-security-proving-grounds-gaara-a7247f66720a)
* [Offsec - Gaara - Jun 3rd 2023 | 1337 Sheets](https://quaily.com/prestonzen/p/offsec-gaara-jun-3rd-2023)
* [Gaara - Proving Grounds Writeup | ziomsec](https://ziomsec.com/writeups/misc/gaara/)
* [Gaara Write-Up (Provinggrounds / Vulnhub) | by James Jarvis - Medium](https://medium.com/@jamesjarviscyber/gaara-write-up-provinggrounds-vulnhub-0168a5beb32e)
