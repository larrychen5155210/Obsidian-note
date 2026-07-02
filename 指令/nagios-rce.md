# Nagios XI 後台 RCE 與提權 (nagios-rce)
本文件說明 Nagios XI 監視系統的權限劫持漏洞利用手法。

## 💥 漏洞介紹
*   **CVE 編號**：CVE-2019-15949 (適用於 Nagios XI 5.6.0)
*   **漏洞原理**：後台下載系統 Profile 會以 root 權限免密碼執行 `getprofile.sh`，該腳本在運作中會調用低權限 `nagios` 使用者可寫的監控插件 `check_plugin`。

## 🛠️ 利用流程
1.  利用弱密碼或預設憑證（如 `nagiosadmin:admin`）登入 Nagios 後台。
2.  在管理介面中，將 Reverse Shell 惡意指令寫入或替換至 `check_plugin` 插件中。
3.  進入系統資訊下載 Profile 檔案，後台會以 root 權限調用該插件，進而觸發 Reverse Shell，直接取得 `root` 權限的 Shell。

---

# 實戰關聯
*   **靶機應用實例**：[[Monitoring]]
