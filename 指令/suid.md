# SUID 權限服務劫持 (suid)
本文件說明如何利用 SUID 權限進行本地特權提升，並以 Serv-U FTP Server 漏洞為例。

## 🔍 檢測本機 SUID 檔案
可以使用以下指令枚舉系統中所有具備 SUID 權限的二進位檔案：
```bash
find / -perm -u=s -type f 2>/dev/null
```

## 💥 Serv-U FTP Server 本地提權 (serv-u-ftp-server)
*   **CVE 編號**：CVE-2019-12181 (適用於 Serv-U FTP Server < 15.1.7)
*   **利用方法**：
    透過本機資訊枚舉（如 `linpeas`）發現具有 SUID 屬性的 `/usr/local/Serv-U`，使用對應的 Exploits 本地利用程式：
    ```bash
    # 賦予 Exploits 腳本執行權限並運行
    chmod +x 47173.sh
    ./47173.sh
    ```
    執行成功後，將會自動釋放一個 root SUID Shell 位於 `/tmp/sh`，執行它即可獲得 `root` 權限。

---

# 實戰關聯
*   **靶機應用實例**：[[Election1]]
