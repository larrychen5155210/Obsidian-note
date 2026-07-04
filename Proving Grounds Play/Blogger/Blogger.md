---
title: Blogger 簡介
date: 2026-07-04
tags:
  - pg-play
  - linux
  - wordpress
  - wpdiscuz
  - file-upload
  - pwnkit
status: completed
difficulty: easy
---

# Blogger 簡介

本筆記摘要自詳細報告 [[Blogger-Walkthrough]]。以下是該靶機的快速滲透路徑：

## 1. 偵察與列舉 (Reconnaissance)
*   **連接埠掃描**：開放 SSH (22) 與 HTTP (80)。
*   **網頁目錄與服務列舉**：
    *   目錄爆破發現 `/assets/fonts/blog/` 底下架設了 WordPress 網站。
    *   在 `/etc/hosts` 中加入 `blogger.pg` 以確保網站資源能正確載入。
    *   使用 `wpscan` 進行使用者列舉，發現 `j@m3s` 用戶，並確認 `xmlrpc.php` 接口處於啟用狀態。

## 2. 初始存取 (Initial Access)
*   **[[wordpress#💥 wpDiscuz 7.0.4 未授權任意檔案上傳 (CVE-2020-24186)|wpDiscuz 檔案上傳漏洞]]**：
    *   WordPress 站台啟用有漏洞的 `wpDiscuz 7.0.4` 留言板外掛，允許使用者在評論時上傳圖片（CVE-2020-24186）。
    *   在上傳的 PHP Web Shell 檔案最前端加上 [[file-upload#💥 Magic Number 與文件簽名繞過|Magic Number GIF89a; 簽名]] 繞過後端的檔案類型（MIME/Signature）檢查，成功上傳 `shell.php`。
    *   發布評論後，點擊或存取上傳的檔案路徑以觸發 Reverse Shell，順利取得 `www-data` 的初始存取權限。

## 3. 特權提升 (Privilege Escalation)
*   **MySQL 資料庫與 Hash 提取**：
    *   在 `wp-config.php` 設定檔中發現資料庫密碼，隨後利用 [[mysql#🔍 MySQL 本地列舉與資料庫提取 (WordPress 例)|MySQL 本地列舉]] 連結並 dump 出 `wordpress.wp_users` 的密碼 Hash（使用 `hashcat -m 400` 進行破解）。
*   **切換 Vagrant 用戶提權 (非預期途徑)**：
    *   由於該靶機是基於 Vagrant 建立的 VM，發現系統中存在 `vagrant` 用戶且使用預設密碼 `vagrant`。
    *   在 shell 中執行 `su vagrant`（密碼 `vagrant`）成功切換用戶。
    *   執行 `sudo -l` 發現 `vagrant` 用戶擁有 NOPASSWD 的 root 執行權限，隨後執行 `sudo su -` 成功提權至 `root` 取得 `proof.txt`。
*   **PwnKit 本地提權 (核心漏洞 - 替代路徑)**：
    *   由於該系統 Polkit 版本較舊，亦可直接使用 [[pwnkit### 🛠️ 利用方法|PwnKit (CVE-2021-4034)]] 漏洞。
    *   將漏洞利用程式 `PwnKit` 上傳至目標主機並執行，即可直接取得 `root` 權限。

---

## 🔗 資料來源 (References)
* [Aslam Mahimkar Medium Writeup](https://medium.com/@aslam.mahimkar/oscp-proving-ground-play-blogger-1-writeup-731a1330d8c1)
* [CyberArri Blogger Walkthrough](https://cyberarri.com/2024/03/17/blogger-pg-play-writeup/)
* [Pika5164 GitHub Writeup](https://github.com/pika5164/Offsec_Proving_Grounds/blob/master/PG_Play/Blogger.md)
* [Hamza Malick Medium Walkthrough](https://medium.com/@hamzamalick72/blogger-walkthrough-proving-ground-oscp-prep-2638e865304d)
* [S1REN YouTube Walkthrough](https://www.youtube.com/watch?v=cv8CcwtXL3M)
* [Cyb3r-0verwatch YouTube Walkthrough](https://www.youtube.com/watch?v=TlwivZlByeg)
