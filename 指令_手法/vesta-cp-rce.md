# 💥 Vesta Control Panel 認證後遠端代碼執行 (vesta-cp-rce)

本文件說明 Vesta Control Panel (VestaCP) 管理面板認證後命令注入與遠端權限提升漏洞的利用方法。

---

## 🔍 漏洞成因與原理

**Vesta Control Panel (VestaCP)** 是一款開源的 Linux 主機與網站管理面板，預設在 **8083 端口** 運行。

在 VestaCP 的郵件管理模組中，允許登入用戶建立與編輯電子郵件網域與信箱。然而，後端代碼在處理郵件信箱的配置更新時，沒有對使用者輸入的特定欄位進行適當的過濾，隨後將其直接串接並帶入以 **`root`** 權限執行的後台 Shell 命令（如執行重啟郵件服務或配置同步之腳本）。

攻擊者在取得 VestaCP 的普通用戶帳密並登入後，可透過精心構造的信箱編輯封包注入惡意命令。由於執行該命令的後端守護進程是以最高權限運行的，命令會直接以 `root` 身份執行，導致普通用戶能藉此獲得目標主機的 root 權限 Shell。

---

## 🛠️ 常見利用與繞過技巧

### 1. 偵測與登入
*   **確認 VestaCP 運行**：
    網頁訪問 `https://<TARGET_IP>:8083`。
*   **認證憑證登入**：
    輸入已知的普通用戶憑證（如藉由 SNMP 洩漏轉儲獲得的密碼）完成登入。

---

### 2. 基本利用方法 (Python Exploit)
通常使用公開的命令注入 POC 腳本來自動化此過程（自動建立惡意網域、郵箱，編輯觸發命令，並反彈 Shell）：

*   **執行 RCE 腳本**：
    ```bash
    python3 vesta-rce-exploit.py https://<TARGET_IP>:8083 <USERNAME> <PASSWORD>
    ```
    *腳本在執行後，會回傳一個交互式 Shell 輸入介面，此時輸入指令，將直接以 root 權限執行：*
    ```bash
    [INFO] Attempting login to https://192.168.173.156:8083 as jack
    [+] Logged in as jack
    [+] Mail domain and mailbox created for backdoor deployment
    [INFO] Deploying backdoor via mailbox editing
    [+] Root shell possibly obtained. Enter commands:
    # whoami
    root
    ```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[OSCP-C]]（在 Linux 外網主機 192.168.x.156 滲透中，攻擊者先透過 SNMPwalk 漏洞竊取了 `jack` 的密碼 `3PUKsX98BMupBiCf`，並登入 8083 Port 的 VestaCP。隨後在 Kali 本地執行 `vesta-rce-exploit.py` 腳本，提供 jack 的帳密，利用該漏洞成功以 root 權限在後台執行指令，直接獲取目標的最高系統 root 權限）。
