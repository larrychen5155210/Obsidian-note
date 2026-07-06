# 💥 連接埠敲擊 (port-knocking)

本文件說明連接埠敲擊 (Port Knocking) 網絡安全防護機制的運作原理、檢測方法與滲透測試中的常用敲擊手段。

---

## 🔍 漏洞成因與原理

連接埠敲擊 (Port Knocking) 是一種藉由在一組預先定義好的關閉連接埠上，依特定順序發送嘗試連接（Knock）來動態開啟防火牆規則（例如打開原本被隱藏 Filtered 的 SSH 22 埠）的機制。
其服務端常使用 `knockd` 守護行程（Daemon）來監聽網卡上的特定連線嘗試：
*   當檢測到符合設定檔 `/etc/knockd.conf` 中的特定敲擊順序（例如依次連接 7469 -> 8475 -> 9842 TCP 連接埠）時，便會執行對應的系統指令（通常是 `iptables` 規則插入）以允許敲擊來源 IP 的訪問。
*   通常也設有 closeSSH 的相反順序或超時自動關閉的機制。

在滲透測試中，若發現某些服務被過濾（Filtered），但具備讀取檔案（LFI/SQLi 讀取）的權限，可透過查閱配置檔來取得敲擊序列。

---

## 🛠️ 常見利用與繞過技巧

### 1. 使用專用敲擊工具 (knock)
若攻擊機已安裝 `knock` 工具，可直接執行敲擊：
```bash
# 依次敲擊 TCP 序列 (預設為 TCP)
knock -v <TARGET_IP> 7469 8475 9842

# 若敲擊序列包含 UDP 埠，可加上 -u 參數
knock -v <TARGET_IP> 7469:udp 8475:tcp 9842:udp
```

### 2. 使用 Nmap 進行敲擊
在沒有 `knock` 工具的環境下，可利用 `nmap` 對各個埠進行極快速且不進行重試的掃描來達到敲擊效果：
```bash
# 使用 Bash 迴圈搭配 nmap 進行敲擊
for port in 7469 8475 9842 ; do nmap -Pn --max-retries 0 -p$port <TARGET_IP> ; done
```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[DC-9]]（對網頁服務進行目錄走訪讀取 `/etc/knockd.conf` 取得敲擊序列 `7469, 8475, 9842`，利用 nmap 依次敲擊後，成功將 Filtered 狀態的 SSH (Port 22) 轉為 Open 狀態以進行登入）。
