# 💥 AD 域內 ACL 權限濫用 (ad-acl-abuse)

本文件說明在 Active Directory 環境中，當域帳號對其他域對象（如 User、Computer）擁有過大或不當的存取控制清單（DACL）權限（例如 `GenericWrite`、`GenericAll`、`WriteDacl` 等）時，常見的權限濫用與提權利用手法。

---

## 🔍 漏洞成因與原理

在 Active Directory 域環境中，每個物件（如使用者、電腦、安全群組等）都擁有一個安全描述項（Security Descriptor），其中包含 **隨意存取控制清單 (DACL, Discretionary Access Control List)**。DACL 由多個 **存取控制項目 (ACE, Access Control Entry)** 組成，用來界定域帳戶對該物件所擁有的權限。

如果某些普通域帳戶被賦予了過大的權限（例如委派設定錯誤），攻擊者即可透過濫用這些 DACL 權限來控制目標對象，進而完成域內橫向移動或特權提升。其中最常見的權限包括：
*   **GenericWrite**：通用寫入權限，允許寫入該物件的任何非受保護屬性（如修改 SPN、重設某些屬性）。
*   **GenericAll**：完全控制權限，包含修改密碼、寫入屬性、更改 ACL 等所有權限。
*   **WriteDacl**：寫入 DACL 權限，允許修改該物件的 DACL 設定，藉此將自己加入完全控制（GenericAll）名單。
*   **WriteOwner**：修改所有者權限，允許將物件的所有權人改為自己，進而取得控制權。

---

## 🛠️ 常見利用與繞過技巧

### 💥 GenericWrite 權限對使用者對象之濫用 (Targeted Kerberoasting)
當擁有對另一個使用者的 `GenericWrite` 權限時，攻擊者可以透過為其註冊一個虛擬的 **服務主體名稱 (SPN, Service Principal Name)**，強行將該普通使用者帳戶轉變為可受 Kerberoasting 攻擊的對象。隨後對該對象發起 **Targeted Kerberoasting** 請求 TGS 票據雜湊值並離線破解以獲取其明文密碼。

#### 利用方法
1.  **使用 BloodHound 分析 ACL**：
    在 BloodHound 中確認當前用戶（如 `yulia.weber`）對另一高權限使用者（如 `boris.crawford`）擁有 `GenericWrite` 權限。
2.  **利用 [targetedKerberoast.py](https://github.com/ShutdownRepo/targetedKerberoast/tree/main) 自動完成攻擊**：
    該工具會自動使用當前憑證為目標用戶註冊 SPN，請求 TGS 票據，並隨後清除該 SPN 以免留下痕跡：
    ```bash
    python3 targetedKerberoast.py -d <DOMAIN> -u <username> -p '<password>' --request-user <target_user> --dc-ip <DC_IP> -o <output_hash_file> --only-abuse
    ```
3.  **離線破解雜湊值**：
    詳細的離線 Hash 破解指令可參考 [[hash-cracking#3. Kerberos TGS-REP 破解 (Mode 13100)|密碼與雜湊值破解 - Kerberos TGS-REP 破解]]，使用 `hashcat` 模式 `13100` 進行字典爆破。

---

### 💥 AllExtendedRights 權限對使用者對象之濫用 (Force Change Password)
當擁有對另一個使用者的 `AllExtendedRights`（所有擴充權限）或 `ForceChangePassword` 權限時，攻擊者可以不需知道目標使用者的舊密碼，直接透過 LDAP 強制重設該使用者的密碼為新密碼。

#### 利用方法
1.  **使用 BloodHound 分析 ACL**：
    在 BloodHound 中確認當前用戶（如 `lisa`）對另一域使用者（如 `jackie`）擁有 `AllExtendedRights` 權限。
2.  **強制修改目標使用者密碼**：
    利用 Linux 系統上的 `net` 工具強制重設目標使用者的密碼：
    ```bash
    # 重設密碼為 "password"
    net rpc password "<target_user>" "<new_password>" -U "<DOMAIN>"/"<username>"%"<password>" -S "<DC_IP>"
    ```
3.  **使用新密碼登入或進行橫向移動**：
    驗證新密碼是否可透過 WinRM 或 SMB 進行登入利用。

---

### 💥 GenericAll 權限對使用者對象之濫用 (Force Change Password)
當擁有對另一個使用者的 `GenericAll`（完全控制）權限時，攻擊者擁有對該對象的完整控制權，包括重設其密碼。攻擊者可以直接在已登入域的 Shell 中使用 `net user` 指令強制修改其密碼為新密碼。

#### 利用方法
1.  **使用 BloodHound 分析 ACL**：
    在 BloodHound 中確認當前用戶（如 `z.thomas`）對另一域使用者（如 `d.chambers`）擁有 `GenericAll` 權限。
2.  **強制修改目標使用者密碼**：
    在已控制的域成員主機 Shell 中，執行以下指令將目標使用者的密碼強制重設為新密碼：
    ```powershell
    # 重設密碼為 "password"
    net user <target_user> <new_password> /domain
    ```
3.  **使用新密碼登入利用**：
    驗證新密碼是否可透過 WinRM 或 SMB 進行登入利用。

---

### 💥 GenericWrite 權限對電腦對象之濫用 (RBCD)
當擁有對某台電腦帳戶的 `GenericWrite` 權限時，攻擊者可以修改該電腦物件的 `msDS-AllowedToActOnBehalfOfOtherIdentity` 屬性，設定 **資源基於委派的约束性委派 (RBCD, Resource-Based Constrained Delegation)**。這允許攻擊者控制的電腦帳戶（或新增的機器帳戶）代表任何域使用者（包括 Domain Admin）向該目標電腦申請服務票據，進而接管該電腦主機。

#### 利用方法
1.  **利用已控帳戶或機器帳戶寫入委派**：
    使用 `impacket-rbcd` 來設定委派關係，讓攻擊者控制的機器帳戶（如 `KALI-FAKE$`）能代表任何用戶向目標電腦進行驗證：
    ```bash
    impacket-rbcd -delegate-from 'KALI-FAKE$' -delegate-to '<target_computer_name>' -action 'write' -dc-ip <DC_IP> <DOMAIN>/<username>:<password>
    ```
2.  **申請 impersonate 服務票據**：
    利用 `getST.py` 模擬域管理員 `Administrator` 獲取針對目標電腦服務的 Ticket：
    ```bash
    impacket-getST -spn 'cifs/<target_computer_name>.<domain>' -impersonate 'Administrator' -dc-ip <DC_IP> <DOMAIN>/'KALI-FAKE$':'<fake_computer_password>'
    ```
3.  **使用 Ticket 登入目標電腦**：
    匯入票據（設定環境變數 `KRB5CCNAME`）後，使用 `secretsdump` 或 `psexec` 登入目標電腦：
    ```bash
    export KRB5CCNAME=<ticket_file>.ccache
    impacket-secretsdump -k -no-pass <target_computer_name>.<domain>
    ```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Laser]]（域帳戶 `yulia.weber` 對域帳戶 `boris.crawford` 具有 `GenericWrite` 權限。使用 `targetedKerberoast.py` 自動為其設定 SPN 並發起 Targeted Kerberoasting，導出 Ticket Hash 後，利用 `hashcat` 成功破解出 `boris.crawford` 的明文密碼 `zxcvbnm`，成功接管該高權限管理員帳號）。
    *   [[Poseidon]]（域帳戶 `lisa` 對域帳戶 `jackie` 具有 `AllExtendedRights` 權限。使用 `net rpc password` 強行修改 `jackie` 的密碼為 `password`，隨後成功透過 WinRM 登入 `GYOZA` 主機取得其 Shell 存取權限）。
    *   [[Zeus]]（域帳戶 `z.thomas` 對域帳戶 `d.chambers` 具有 `GenericAll` 權限。在已登入的 `DC01` 上執行 `net user d.chambers password /domain` 強制將 `d.chambers` 的密碼重設為 `password`，隨後成功透過 WinRM 登入 `DC01` 控制器）。
