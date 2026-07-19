---
title: "OSCP C 簡介"
date: "2026-07-19"
tags:
  - oscp-lab
  - active-directory
  - windows
  - linux
  - rce
  - service-binary-replacement
  - snmp
  - tar-wildcard
  - hash-cracking
status: completed
difficulty: hard
---

# OSCP C 簡介

本筆記摘要自詳細報告 [[OSCP-C-Walkthrough]]。這是一個涉及多台外網獨立主機與內網 AD 網域結構的綜合滲透測試模擬環境，以下是該環境的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **外網主機與連接埠掃描**：
    *   `192.168.x.153 (MS01 - Windows / AD 主機)`：開啟 22, 135, 139, 445, 5985, 8000 Port。
    *   `192.168.x.155 (Windows 獨立主機)`：開啟 80, 9099 (Mobile Mouse), 9999 Port。
    *   `192.168.x.156 (Linux 獨立主機)`：開啟 21, 22, 25, 53, 80, 110, 143, 8080, 8083 (VestaCP) Port 等。
    *   `192.168.x.157 (Linux 獨立主機)`：開啟 21 (FTP), 22, 80, 20000 (Webmin) Port。
*   **NXC 網域主機列舉**：
    *   使用已知的域帳密 `Eric.Wallows:EricLikesRunning800` 對外網主機進行檢測，確認其擁有 `192.168.x.153` (MS01) 的 WinRM 與 SSH 存取權限。

## 2. 初始存取與外網接管 (Initial Access & DMZ Compromise)
*   **Windows 獨立主機 (192.168.x.155)**：
    *   利用 [[mobile-mouse-rce#💥 Mobile Mouse 服務遠端代碼執行 (mobile-mouse-rce)|Mobile Mouse RCE]] 漏洞配合 Metasploit 模組，成功取得帳戶 `tim` 的初始存取 Shell。
*   **Linux 獨立主機 (192.168.x.156)**：
    *   利用 UDP 161 [[snmp#💥 SNMP 資訊收集與敏感憑證洩漏 (snmp)|SNMPwalk 資訊收集]] 轉儲出 `NET-SNMP-EXTEND-MIB` 擴充參數，竊取 `jack` 的明文密碼：**`3PUKsX98BMupBiCf`**。
    *   登入 8083 Port 的 VestaCP 管理面板後，利用 [[vesta-cp-rce#💥 Vesta Control Panel 認證後遠端代碼執行 (vesta-cp-rce)|VestaCP RCE]] 漏洞利用腳本直接提升至 root 權限，取得 `proof.txt`。
*   **Linux 獨立主機 (192.168.x.157)**：
    *   匿名登入 FTP，下載範本文件並利用 [[exiftool#💥 Exiftool Metadata 敏感資訊洩漏與使用者列舉 (exiftool)|exiftool 讀取 PDF 元數據]] 提取出作者帳密字典，發現弱密碼 `cassie:cassie` 並成功登入 20000 Port 的 Webmin 管理介面，反彈取得 `cassie` 權限 Shell。

## 3. 本地特權提升 (Privilege Escalation)
*   **192.168.x.155 (Windows)**：
    *   經本地列舉發現普通使用者組對 `GPGService.exe` 具有修改 `(M)` 權限，利用 [[service-binary-replacement#💥 服務二進位檔替換提權 (service-binary-replacement)|服務二進位檔替換提權]] 手法，使用 `msfvenom` 生成同名服務 Shell 並進行覆寫替換，以系統 `SYSTEM` 權限重啟服務完成提權，取得 `proof.txt`。
*   **192.168.x.157 (Linux)**：
    *   發現定時排程以 root 權限執行 `tar *`，且使用者對該執行目錄有寫入權限。利用 [[tar-wildcard#💥 Tar 萬用字元參數注入|Tar 萬用字元參數注入]] 技巧，建立帶有 checkpoint 參數的空檔案，使 tar 在打包時執行自訂腳本賦予 `/bin/bash` SUID，成功執行 `bash -p` 取得 root 權限與 `proof.txt`。

## 4. 內網穿透與網域接管 (Pivoting & Domain Compromise)
*   **外網 AD 進入點 (192.168.x.153 - MS01)**：
    *   對 8000 Port 進行 feroxbuster 目錄列舉，發現 `/partner/db` SQLite 資料庫，下載後使用 sqlite3 導出，並利用 [[hash-cracking#1. MD5 雜湊破解 (Mode 0)|MD5 離線破解]] 爆破出 `support` 的密碼：**`Freedom1`**。
*   **域內橫向移動 (MS01)**：
    *   使用 `Eric.Wallows:EricLikesRunning800` 透過 SSH 登入 Windows 主機 MS01。在 `Eric.Wallows` 用戶目錄下嘗試執行 `admintool.exe`，藉由觸發 panic 資訊或 `strings` 逆向獲取本地管理員明文密碼：**`December31`**。
*   **內網穿透與橫向移動 (MS02)**：
    *   使用 `impacket-secretsdump` 轉儲 MS01 本地 SAM，取得網域停用帳戶 `celia.almeda:7k8XHk3dMtmpnC7`。
    *   利用 [[powershell-history#💥 PowerShell 歷史命令紀錄敏感資訊洩漏 (powershell-history)|PowerShell 歷史命令紀錄讀取]]，在本地管理員的歷史命令中，發現執行 `admintool.exe` 參數中帶有另一組管理員密碼：**`hghgib6vHT3bVWf`**。
    *   架設 `ligolo-ng` 隧道以 MS01 作為跳板進入 `10.10.x.0/24` 內網，使用 `hghgib6vHT3bVWf` 憑證成功以本機管理員權限透過 WinRM 登入 `10.10.133.154` (MS02)。
*   **網域接管 (DC01)**：
    *   在 MS02 主機上使用 `secretsdump` 導出網域憑證，轉儲出系統預設的網域管理員明文密碼：`OSCP.exam\Administrator:7Tg9M9MZbzAokR9`。
    *   最後在 Kali 本地透過 impacket 的 `psexec` 以域管憑證直接登入網域控制器 `10.10.133.152` (DC01)，取得 SYSTEM 權限與最後的 `proof.txt`。

## 🔗 資料來源 (References)
* [OSCP C - HackMD](https://hackmd.io/@5JVAxiEFSz-M2n3hGZoxgQ/SJUr68TxMg)
