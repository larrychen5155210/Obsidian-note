---
title: "Monitoring 簡介"
date: "2026-06-29"
tags:
  - pg-play
  - linux
  - nagios-rce
status: completed
difficulty: easy
---

# Monitoring 簡介

本筆記摘要自詳細報告 [[Monitoring-Walkthrough]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：開啟多個連接埠，包括 `22 (SSH)`、`25 (SMTP)`、`80 (HTTP)`、`389 (LDAP)`、`443 (HTTPS)` 與 `5667 (Nagios)`。
*   **網頁與服務探測**：網頁服務為 **Nagios XI** 監控系統，直接訪問 `http` 連接埠會觸發 NSP 報錯（Sorry Dave, I can't let you do that），但透過 `https` 可正常開啟登入介面。

## 2. 初始存取與特權提升 (Initial Access & Privilege Escalation - 一步到位)
*   **預設憑證登入**：使用 Nagios XI 預設管理員帳密 `nagiosadmin:admin` 成功登入系統。
*   **版本弱點分析**：登入後確認系統版本為舊版 `Nagios XI 5.6.0`。
*   **遠端代碼執行 (RCE) / Root 提權**：
    *   Nagios XI 5.6.0 存在 [[nagios-rce|CVE-2019-15949]] 本地/遠端 Root 提權漏洞。
    *   在系統內下載系統設定檔時，系統會以 `root` 權限執行定時備份/下載指令碼 `getprofile.sh`（免密碼 sudo 權限）。
    *   `getprofile.sh` 在執行過程中會呼叫由 `nagios` 使用者所擁有的 `check_plugin` 檔案。
    *   管理員（或 nagios 使用者）可以透過上傳/修改該 plugin 檔案內容注入惡意指令，當觸發系統設定檔下載時，惡意指令便會以 `root` 權限被執行。
*   **Shell 取得**：使用 `searchsploit` 下載 RCE 腳本 (`52138.py`)，針對 HTTP 服務直接傳入憑證與監聽參數執行，於本機 Netcat 成功取得具備 `root` 權限的 Reverse Shell。
