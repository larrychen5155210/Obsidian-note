---
title: "Zeus 簡介"
date: "2026-07-19"
tags:
  - oscp-lab
  - active-directory
  - windows
  - ad-acl-abuse
status: completed
difficulty: hard
---

# Zeus 簡介

本筆記摘要自詳細報告 [[Zeus-Walkthrough]]。此靶機為包含成員工作站與域控制器的 Active Directory 域環境（`zeus.corp`），以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **網域主機列舉**：使用已知的域帳密 `Eric.Wallows:EricLikesRunning800`，配合 `nxc winrm` 列舉網域主機，發現以下三台主機：
    *   `DC01 (192.168.113.158)`：域控制器。
    *   `CLIENT01 (192.168.113.159)`：成員伺服器，可以使用 WinRM 登入。
    *   `CLIENT02 (192.168.113.160)`：成員工作站。

## 2. 初始存取 (Initial Access)
*   **本機管理員與憑證導出**：
    *   使用 `Eric.Wallows` 帳號透過 WinRM 登入 `CLIENT01`，發現其在本地 `Administrators` 群組中，直接獲取本機管理員 Shell 權限 (取得第一個 `proof.txt`)。
    *   開啟 RDP 服務後，使用 `xfreerdp3` 登入 `CLIENT01` 並執行 `mimikatz`，成功導出記憶體明文憑證，取得域帳戶 `o.foller` 密碼：**`EarlyMorningFootball777`**。

## 3. 橫向移動與資訊收集 (Lateral Movement & Enumeration)
*   **橫向移動工作站與敏感文件洩漏**：
    *   使用 `o.foller` 帳密通過 `nxc` 驗證其可登入 `CLIENT02`，利用 `psexec` 取得 `SYSTEM` 權限 Shell 並獲取第二個 `proof.txt`。
    *   列舉本機檔案時，在 `z.thomas` 使用者的 `Downloads` 目錄下發現 `Onboarding Document.docx` 文件，在文件內容中發現洩漏了域帳戶 `z.thomas` 的明文密碼：**`^1+>pdRLwyct]j,CYmyi`**。

## 4. 域內 ACL 濫用與域控提權 (AD ACL Abuse & DC Escalation)
*   **域控權限委派修改 (GenericAll)**：
    *   使用 `z.thomas` 帳號透過 WinRM 登入域控制器 `DC01`，取得 `local.txt`。
    *   在域控制器上執行 BloodHound 收集資訊，分析結果顯示 `z.thomas` 對域帳戶 `d.chambers` 具有 `GenericAll` 權限。
    *   利用 [[ad-acl-abuse#💥 GenericAll 權限對使用者對象之濫用 (Force Change Password)|GenericAll 權限濫用]] 手法，直接執行 `net user d.chambers password /domain` 強制將其密碼重設為 `password`。
*   **憑證備份特權轉儲 (SeBackupPrivilege)**：
    *   使用 `d.chambers` 帳號以 WinRM 登入 `DC01`（域控制器），發現其擁有 `SeBackupPrivilege` 特權。
    *   利用 [[sebackupprivilege#💥 SeBackupPrivilege 本地提權與憑證轉儲 (sebackupprivilege)|SeBackupPrivilege 憑證轉儲]] 手法，使用 Windows 內建的 `diskshadow` 工具執行快照腳本，將磁碟快照掛載至 `X:\`，隨後配合 `robocopy /B` 複製出 `ntds.dit` 和 `SYSTEM` 檔案。
    *   在 Kali 端使用 `secretsdump` 進行離線解密，取得 `Administrator` 的 NTLM Hash，最後使用 `psexec` 接管整個域控 `DC01`，取得最後域控的 `proof.txt`。

## 🔗 資料來源 (References)
* [Zeus - HackMD](https://hackmd.io/@5JVAxiEFSz-M2n3hGZoxgQ/S1tF7dOZGg)
