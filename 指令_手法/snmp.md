# 💥 SNMP 資訊收集與敏感憑證洩漏 (snmp)

本文件說明 SNMP 服務的資訊收集、遞迴列舉方法，以及如何藉此發現系統中殘留的敏感資訊或明文憑證。

---

## 🔍 漏洞成因與原理

**SNMP (Simple Network Management Protocol，簡單網路管理協定)** 廣泛用於網路設備與伺服器的監控與管理。它使用 **社群字串 (Community String)** 作為基本的身份驗證機制。在許多環境中，管理員會忘記變更預設的唯讀社群字串（例如 `public`）或讀寫社群字串（例如 `private`）。

如果 SNMP 服務暴露於外網，且使用弱密碼或預設社群字串，攻擊者便可透過 SNMP 查詢（如使用 `snmpwalk`）遞迴列舉出系統的敏感資訊，包含：
*   系統進程與啟動參數（常含有敏感腳本或明文密碼參數）。
*   系統安裝的軟體與版本。
*   網路配置與路由表。
*   **擴充 MIB 物件 (如 `NET-SNMP-EXTEND-MIB`)**：在 Net-SNMP 中，管理員可設定擴充腳本 (Extend commands)。這些擴充命令的執行參數（如修改密碼的 `chpasswd` 命令參數）會被永久記錄在擴充物件的 MIB 中，若遭到 SNMP 查詢，將直接洩漏明文憑證。

---

## 🛠️ 常見利用與繞過技巧

### 1. 服務掃描與社群字串爆破
SNMP 主要運行在 UDP 161 端口。
*   **使用 Nmap 偵測 UDP 連接埠**：
    ```bash
    nmap -Pn -sU -p 161 <TARGET_IP>
    ```
*   **使用 Onesixone 爆破社群字串**：
    ```bash
    onesixone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <TARGET_IP>
    ```

---

### 2. 基本利用方法 (列舉與轉儲)
一旦確認社群字串為 `public`，即可使用 `snmpwalk` 進行資訊列舉：

*   **列舉所有系統進程命令行 (Process Parameters)**：
    ```bash
    # 查詢 hrSWRunParameters (1.3.6.1.2.1.25.4.2.1.5)
    snmpwalk -v 2c -c public <TARGET_IP> 1.3.6.1.2.1.25.4.2.1.5
    ```
*   **查詢 Net-SNMP 擴充指令記錄 (NET-SNMP-EXTEND-MIB)**：
    此處極易洩漏系統腳本執行的命令與參數：
    ```bash
    snmpwalk -v 2c -c public <TARGET_IP> NET-SNMP-EXTEND-MIB::nsExtendObjects
    ```
    *例如：在輸出中發現 `nsExtendArgs` 包含 `echo "jack:3PUKsX98BMupBiCf" | chpasswd`，即代表成功取得用戶 jack 的明文憑證。*

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[OSCP-C]]（在列舉 Linux 靶機時，發現 192.168.x.156 開啟了 UDP 161 SNMP 服務。使用 `snmpwalk` 配合 `public` 社群字串查詢 `NET-SNMP-EXTEND-MIB` 擴充組件，成功轉儲出系統重設密碼腳本的殘留參數，洩漏了本機用戶 `jack` 的明文密碼：`3PUKsX98BMupBiCf`，進而成功登入管理後台）。
