---
title: "Beyond 簡介"
date: "2026-07-15"
tags:
  - oscp-lab
  - active-directory
  - multi-host
  - linux
  - windows
  - wordpress
  - arbitrary-file-read
  - path-traversal
  - ssh-private-key
  - hash-cracking
  - sudo-exploitation
  - phishing
  - kerberoasting
  - ntlm-relay
status: completed
difficulty: hard
---

# Beyond 簡介

本筆記摘要自詳細報告 [[Beyond-Walkthrough]]。此靶機為包含多台 Linux 及 Windows 主機的 Active Directory 域（Beyond.com）環境，以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **外網服務偵察**：
    *   `192.168.224.242` (Mailserver)：開放 SMTP (hMailServer), IIS (80), POP3, IMAP, SMB (445)。
    *   `192.168.224.244` (Webserver)：開放 SSH (22), Apache httpd (80)。
*   **Web 服務列舉**：使用 `wpscan` 針對 Webserver 的 WordPress 掃描，發現啟用了舊版 `Duplicator 1.3.26` 插件。

## 2. 初始存取 (Initial Access)
*   **漏洞利用 / 憑證獲取**：
    *   利用 [[wordpress#💥 Duplicator 1.3.26 未授權任意檔案讀取 (CVE-2020-11738)|Duplicator 任意檔案讀取]] 漏洞讀取 `/etc/passwd`，列舉出使用者 `daniela` 與 `marcus`；進而讀取 `daniela` 的 SSH 私鑰 `/home/daniela/.ssh/id_rsa`。
    *   利用 [[ssh-private-key#💥 破解加密私鑰 (Passphrase Protected Key)|SSH 私鑰破解]] 手法，使用 `ssh2john` 配合 `john` 破解出私鑰 Passphrase 為 `tequieromucho`。
*   **Shell 取得**：使用私鑰與密碼以 SSH 登入 Webserver，取得初始存取權。

## 3. 特權提升與憑證還原 (Privilege Escalation)
*   **Linux 本地提權**：
    *   執行 `sudo -l` 發現 `/usr/bin/git` 具有免密碼 Sudo 權限。
    *   由於 `/srv/www/wordpress/.git` 屬於 root，使用 `sudo git log` 查看歷史提交，還原出已被刪除的腳本並發掘 staging server 的 SSH 憑證 **`john:dqsTwTpZPn#nL`**。
    *   利用 [[sudo#💥 Sudo Git Pager 提權 (GTFOBins Pager Escape)|Sudo Git Pager 提權]] 逃逸手法，執行 `sudo git branch --help config` 並在 less Pager 中輸入 `!/bin/bash` 成功提權至 Root。
*   **域使用者憑證噴灑**：使用 `nxc smb` 測試已還原的帳密，確認 `john` 是有效的 Windows 域帳號 (`beyond.com\john`)。
*   **Windows 釣魚郵件突破**：
    *   架設 WebDAV 託管惡意 `runme.lnk` 快捷方式。
    *   利用 [[phishing#💥 釣魚郵件與惡意檔案初始存取 (phishing)|釣魚郵件]] 技術，使用 `john` 的憑證寄送夾帶 `openme.library-ms` 的信件給內部員工，強迫連線 WebDAV。
    *   使用者雙擊執行後，成功接回反彈 Shell，取得 `CLIENTWK1` 主機的 `beyond\marcus` 權限。

## 4. 內網域環境橫向移動 (Lateral Movement & Pivot)
*   **內網偵察與代理**：
    *   執行 `winpeas` 發現網域控制器 DCSRV1 (`172.16.180.240`) 及內網郵件伺服器 MAILSRV1 (`172.16.180.254`)。
    *   使用 SharpHound 收集域資料並使用 BloodHound 分析，確認 `beccy` 為 Domain Admin。
    *   架設 `Ligolo-NG` 建立轉發隧道代理。
*   **Kerberoasting 攻擊**：發起 [[kerberoasting#💥 域內 Kerberoasting 攻擊 (kerberoasting)|Kerberoasting]]，請求到 `http/internalsrv1.beyond.com` 服務帳戶 `daniela` 的 TGS，破解出明文密碼 **`DANIelaRO123`**。
*   **NTLM 中間人轉發 (Relay)**：
    *   使用 `daniela` 憑證登入 `internalsrv1.beyond.com` 的 WordPress 後台。
    *   偵測發現 MAILSRV1 未強制啟用 SMB 簽名。
    *   利用 [[wordpress#💥 Backup Migration 外掛配合 NTLM 中間人轉發 (Backup Migration NTLM Relay)|Backup Migration NTLM Relay]] 手法，將備份路徑修改為攻擊機的 SMB 共用路徑以誘發連線。
    *   利用 [[ntlm-relay#💥 NTLM 中間人轉發攻擊 (ntlm-relay)|NTLM Relay]] 將認證轉發至 MAILSRV1，成功以 `nt authority\system` 權限接回反彈 Shell。

## 5. 域控拿下 (DC Compromise)
*   **憑證轉儲**：在 MAILSRV1 上利用 `mimikatz` 導出記憶體憑證，取得 Domain Admin `beccy` 的 NTLM Hash：**`f0397ec5af49971f6efbdb07877046b3`**。
*   **域控制器突破**：使用 `beccy` 的 NTLM Hash，利用 `impacket-psexec` 登入網域控制器 DCSRV1，奪取最高權限。

---

## 🔗 資料來源 (References)
* [Beyond(27. Assembling the Pieces) - HackMD](https://hackmd.io/@5JVAxiEFSz-M2n3hGZoxgQ/SJrWOl6bfe)
