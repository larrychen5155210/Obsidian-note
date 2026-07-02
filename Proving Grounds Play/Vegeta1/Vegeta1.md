---
title: "Vegeta1 簡介"
date: "2026-07-02"
tags:
  - pg-play
  - linux
  - morse-code
  - writable-etc-passwd
status: completed
difficulty: easy
---

# Vegeta1 簡介

本筆記摘要自詳細報告 [[Vegeta1-Walkthrough]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：確認開放連接埠 `22 (SSH)` 與 `80 (HTTP)`。
*   **服務與目錄列舉**：使用目錄爆破工具發現隱藏路徑 `/bulma/`。
*   **敏感檔案發現**：在 `/bulma/` 目錄下發現一個摩斯密碼音訊檔案 `hahahaha.wav`。

## 2. 初始存取 (Initial Access)
*   **漏洞利用 / 憑證獲取**：將音訊檔下載至本地，使用音訊解密工具解密摩斯密碼，成功獲取 SSH 憑證：`trunks:u$3r`。
*   **Shell 取得**：透過 SSH 使用 `trunks` 帳密成功登入系統，取得初始互動式 Shell（`local.txt`）。

## 3. 特權提升 (Privilege Escalation)
*   **權限提升向量**：於本地枚舉中，發現系統密碼檔 `/etc/passwd` 設定錯誤，存在 [[writable-etc-passwd|可寫系統密碼檔]] 提權漏洞。
*   **取得 Root**：
    *   在本地使用 `openssl` 生成一個加密的密碼 Hash。
    *   將一個具備 root 權限 (UID 0/GID 0) 的自定義使用者追加寫入 `/etc/passwd` 檔案中。
    *   使用 `su` 切換為該自訂使用者，直接提權為 `root` 取得 `proof.txt`。
