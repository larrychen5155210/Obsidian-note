---
title: "Laser 簡介"
date: "2026-07-19"
tags:
  - oscp-lab
  - active-directory
  - windows
  - ntlm-relay
  - kerberoasting
  - ad-acl-abuse
status: completed
difficulty: hard
---

# Laser 簡介

本筆記摘要自詳細報告 [[Laser-Walkthrough]]。此靶機為包含多台 Windows 主機的 Active Directory 域（laser.com）環境，以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠與網域主機列舉**：使用已知的域帳密 `Eric.Wallows:EricLikesRunning800`，配合 `nxc smb` 列舉網域主機，發現域內有三台主機：
    *   `DC01 (192.168.224.172)`：網域控制器，強制啟用 SMB 簽章 (`signing:True`)。
    *   `MS01 (192.168.224.173)`：未啟用 SMB 簽章 (`signing:False`)。
    *   `MS02 (192.168.224.174)`：未啟用 SMB 簽章 (`signing:False`)。
*   **SMB 共享目錄權限列舉**：列舉 shares 發現該帳戶對 `MS01` 上的 `Apps` 共享目錄具有 `READ,WRITE` 寫入權限。

## 2. 初始存取 (Initial Access)
*   **漏洞利用 / 憑證獲取**：
    *   利用 [[ntlm-relay#💥 LNK 惡意檔案 NTLM 憑證洩漏 (NTLM Theft)|LNK NTLM 憑證洩漏]] 手法，使用 `ntlm_theft.py` 生成包含惡意 UNC 路徑的 `clickme.lnk` 檔案，並上傳至 `MS01` 的 `Apps` 共享資料夾。
    *   利用 [[ntlm-relay#💥 Impacket ntlmrelayx 轉發利用 (ntlmrelayx)|ntlmrelayx 轉發利用]] 手法，在 Kali 啟動 `ntlmrelayx` 中繼轉發接收，當受害者開啟目錄觸發 NTLMv2 驗證連線後，將其 Relay 至禁用了 SMB 簽名的 `MS02` 主機。
*   **Shell 取得**：中繼驗證成功後，於 `ntlmrelayx` 中成功建立 `MS02` 主機的 SOCKS 代理通道。透過 proxychains 搭配 `impacket-secretsdump` 導出 `MS02` 的 SAM 哈希值，再利用 `impacket-psexec` 登入 `MS02` (取得 `proof.txt`)。

## 3. 特權提升與橫向移動 (Privilege Escalation & lateral Movement)
*   **封包分析與憑證還原**：
    *   在 `MS02` 的 `Documents` 目錄下發現 `traffic-capture-latest.pcapng` 封包檔案。
    *   將封包下載回 Kali 並使用 Wireshark 進行分析，搜尋關鍵字 `password`，還原出明文帳密憑證：**`yulia.weber:Yulia@Laser777`**。
*   **橫向移動 RDP 登入**：使用驗證出的 `yulia.weber` 憑證，以 RDP 登入網域控制器 `DC01` (取得 `local.txt`)。
*   **域控拿下 (DC Compromise)**：
    *   在 `DC01` 上使用 `bloodhound-python` 進行域資訊收集，BloodHound 分析結果顯示 `yulia.weber` 對域管理員 `boris.crawford` 擁有 `GenericWrite`（通用寫入）權限。
    *   利用 [[ad-acl-abuse#💥 GenericWrite 權限對使用者對象之濫用 (Targeted Kerberoasting)|GenericWrite 權限濫用]] (Targeted Kerberoasting) 手法，使用 `targetedKerberoast.py` 自動為其註冊 SPN 並請求 TGS 票據。
    *   取得 Ticket Hash 後，利用 `hashcat` 成功破解出 `boris.crawford` 的密碼：**`zxcvbnm`**。
    *   因 `boris.crawford` 為全域主機的 Administrator，最終使用該憑證透過 `impacket-psexec` 接管網域控制器 `DC01`，取得 SYSTEM Shell (取得 `proof.txt`)。

## 🔗 資料來源 (References)
* [Laser - HackMD](https://hackmd.io/@5JVAxiEFSz-M2n3hGZoxgQ/Skf6LdjZGl)
