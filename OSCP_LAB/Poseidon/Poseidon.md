---
title: "Poseidon 簡介"
date: "2026-07-19"
tags:
  - oscp-lab
  - active-directory
  - windows
  - hash-cracking
  - ad-acl-abuse
status: completed
difficulty: hard
---

# Poseidon 簡介

本筆記摘要自詳細報告 [[Poseidon-Walkthrough]]。此靶機為包含多網域（子網域 `sub.poseidon.yzx` 與父網域 `poseidon.yzx`）的 Active Directory 森林環境，以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠與網域主機列舉**：使用已知的域帳密 `Eric.Wallows:EricLikesRunning800`，配合 `nxc winrm` 列舉網域主機，發現以下三台主機：
    *   `DC01 (192.168.224.161)`：父網域控制器。
    *   `DC02 (192.168.224.162)`：子網域控制器。
    *   `GYOZA (192.168.224.163)`：子網域成員伺服器，可以使用 WinRM 登入。

## 2. 初始存取 (Initial Access)
*   **WinRM 登入與令牌提權**：
    *   使用 `Eric.Wallows` 帳號透過 WinRM 登入 `GYOZA`，發現當前使用者擁有 `SeImpersonatePrivilege` 權限。
    *   使用 [[seimpersonateprivilege|PrintSpoofer]] 進行本地提權，成功取得 `nt authority\system` Shell (取得 `local.txt` 與 `proof.txt`)。
*   **憑證獲取與橫向移動**：
    *   在 `GYOZA` 上發起 [[as-rep-roasting#💥 域內 AS-REP Roasting 攻擊 (as-rep-roasting)|AS-REP Roasting]] 攻擊，使用 `impacket-GetNPUsers` 請求域帳戶 `chen` 的 TGS，並使用 `hashcat` 破解出其密碼為 `freedom`。
    *   使用 `mimikatz` 導出記憶體憑證，取得域帳戶 `lisa` 的密碼：**`LisaWayToGo456`**。

## 3. 特權提升與橫向移動 (Privilege Escalation & Lateral Movement)
*   **域內 ACL 權限濫用**：
    *   在 `DC02` 上執行 BloodHound 收集資訊，BloodHound 分析結果顯示域帳戶 `lisa` 對高權限域帳戶 `jackie` 擁有 `ALLExtendedRights`（所有擴充權限）。
    *   利用 [[ad-acl-abuse#💥 AllExtendedRights 權限對使用者對象之濫用 (Force Change Password)|AllExtendedRights 權限濫用]] 手法，使用 `net rpc password` 將 `jackie` 的密碼強制重設為 `password`。
*   **憑證備份權限提取 NTDS.dit**：
    *   使用 `jackie` 帳號以 WinRM 登入 `DC02`（子網域控制器），發現其擁有 `SeBackupPrivilege` 特權。
    *   利用 [[sebackupprivilege#💥 SeBackupPrivilege 本地提權與憑證轉儲 (sebackupprivilege)|SeBackupPrivilege 憑證轉儲]] 手法，使用 `vshadow.exe` 建立磁碟快照並掛載，藉由 `robocopy /B` 備份並複製出活動中的子網域 `ntds.dit` 及 `SYSTEM` 檔案。
    *   回傳至 Kali 後使用 `secretsdump` 離線解密，取得子網域 `Administrator` 與 `krbtgt` 的 AES-256 Key。

## 4. 域控拿下 (DC Compromise)
*   **跨域信任關係攻擊 (Child-to-Parent)**：
    *   確認子網域 `sub.poseidon.yzx` 與父網域 `poseidon.yzx` 具有雙向信任關係。
    *   利用 [[child-to-parent-trust#💥 AD 域內跨域信任關係攻擊 (child-to-parent-trust)|Child-to-Parent 跨域信任關係攻擊]] 手法，使用 `impacket-ticketer` 並利用子網域 `krbtgt` 金鑰及父網域企業管理員組的 SID (`-519` Enterprise Admins) 偽造跨域黃金票據 (Golden Ticket)。
    *   Pass-the-Ticket：將票據載入環境變數中，透過 Kerberos 驗證，使用 `impacket-psexec` 登入父網域控制器 `DC01` (192.168.224.161)，成功接管整個 Forest (取得父域控 `proof.txt`)。

## 🔗 資料來源 (References)
* [Poseidon - HackMD](https://hackmd.io/@5JVAxiEFSz-M2n3hGZoxgQ/SJAeTacbze)
