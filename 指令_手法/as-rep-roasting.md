# 💥 域內 AS-REP Roasting 攻擊 (as-rep-roasting)

本文件說明 Active Directory 環境中 AS-REP Roasting 攻擊的原理、漏洞檢測與離線 Hash 破解的技巧。

---

## 🔍 漏洞成因與原理

AS-REP Roasting 是一種針對 Active Directory 域內**未啟用「需要 Kerberos 預先驗證 (Do not require Kerberos preauthentication)」**的帳戶進行密碼哈希離線破解的技術。

在預設情況下，Kerberos 預先驗證會要求使用者在發送身份驗證請求（AS-REQ）時，使用其密碼哈希加密一個時間戳記，以向金鑰分發中心（KDC）證明其身份。
然而，如果某個域帳戶被設定了 **"Do not require Kerberos preauthentication" (UAC 標記 `DONT_REQ_PREAUTH` / `0x400000`)**，任何攻擊者都可以向 KDC 針對該帳號直接發送 AS-REQ 請求。KDC 將會直接回傳包含已使用該帳密哈希加密的票據回覆（AS-REP）。攻擊者即可在本地攔截並提取該加密的票據雜湊值，然後在離線狀態下進行暴力破解，藉此還原該帳戶的明文密碼。

---

## 🛠️ 常見利用與繞過技巧

### 1. 獲取域帳戶雜湊值 (GetNPUsers)
使用 Impacket 工具套件中的 `GetNPUsers.py`，向域控（KDC）請求並下載未啟用預先驗證帳戶的 AS-REP Hash：
```bash
# 以已知的域帳密列舉並請求 Hash (會自動輸出至指定檔案)
impacket-GetNPUsers -dc-ip <DC_IP> -request -outputfile <output_filename.asreproast> <DOMAIN>/<username>:<password>

# 若已知使用者名稱且無憑證，可直接嘗試
impacket-GetNPUsers -dc-ip <DC_IP> -request sub.poseidon.yzx/chen
```

### 2. 離線 Hash 破解
詳細的離線 Hash 破解指令可參考 [[hash-cracking#2. Kerberos AS-REP 破解 (Mode 18200)|密碼與雜湊值破解 - Kerberos AS-REP 破解]]，使用 `hashcat` 模式 `18200`（Kerberos 5 AS-REP etype 23）或 `john` 進行字典爆破。

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Poseidon]]（使用域帳戶 `Eric.Wallows` 的憑證，使用 `impacket-GetNPUsers` 對網域進行 AS-REP Roasting。成功請求到帳戶 `chen` 的 AS-REP Hash，並使用 `hashcat` 模式 18200 破解出密碼 `freedom`，成功擴展內網存取權限）。
