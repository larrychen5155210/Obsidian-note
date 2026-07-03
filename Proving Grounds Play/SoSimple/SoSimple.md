---
title: "SoSimple 簡介"
date: "2026-07-02"
tags:
  - pg-play
  - linux
  - wordpress
  - social-warfare-rce
  - sudo-exploitation
  - ssh-private-key
  - gtfobins
status: completed
difficulty: easy
---

# SoSimple 簡介

本筆記摘要自詳細報告 [[SoSimple-Walkthrough]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：確認開放連接埠 `22 (SSH)` 與 `80 (HTTP)`。
*   **服務與目錄列舉**：爆破目錄發現該站為 WordPress 網站，並使用 `wpscan` 列舉出使用者 `max`。
*   **漏洞發現**：WordPress 中的 `Social Warfare` 外掛版本過舊（< 3.5.3），存在遠端代碼執行 (RCE) 漏洞。

## 2. 初始存取 (Initial Access)
*   **漏洞利用 / 憑證獲取**：
    *   利用 [[wordpress#💥 Social Warfare 遠端代碼執行 (CVE-2019-9978)|Social Warfare RCE]] 漏洞，在攻擊機託管反彈 Shell 的 payload 檔案，觸發漏洞取得 `www-data` 的初始 Shell。
    *   在 `/home/max/.ssh/` 目錄中發現對 `www-data` 可讀的 [[ssh-private-key#🔍 漏洞檢測與憑證發現|id_rsa]] 私鑰檔案。
*   **Shell 取得**：使用該私鑰檔案成功透過 SSH 登入為 `max`，取得互動式 Shell（`local.txt`）。

## 3. 特權提升 (Privilege Escalation)
*   **橫向移動 (max -> steven)**：
    *   執行 `sudo -l` 發現 `max` 可免密碼以 `steven` 權限執行 `/usr/sbin/service` 指令。
    *   利用 [[sudo_-l#💥 Sudo Service 提權|Sudo Service 提權]] 手法，成功切換為使用者 `steven`。
*   **垂直提權 (steven -> root)**：
    *   在 `steven` 權限下執行 `sudo -l` 發現可免密碼以 root 權限執行不存在的 `/opt/tools/server-health.sh` 腳本。
    *   利用 [[sudo_-l#💥 不存在腳本路徑的 Sudo 提權|不存在腳本路徑的 Sudo 提權]] 手法，建立該目錄與腳本並寫入反彈 Shell，執行後取得系統 `root` 權限與 `proof.txt`。

## 🔗 資料來源 (References)
* [Medium - SoSimple CTF Walkthrough](https://medium.com/@wlevi/sosimple-ctf-walkthrough-57049000de08)
* [Medium - SoSimple Walkthrough](https://medium.com/@mahdi_78420/sosimple-walkthrough-6dc88b5b1498)
* [HackMD - SoSimple Walkthrough](https://hackmd.io/@lyws7/BykR5gNzGl)
