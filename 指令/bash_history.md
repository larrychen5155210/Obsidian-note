# 本地命令歷史資訊洩漏 (bash_history)
本文件說明如何利用使用者家目錄權限過大，讀取歷史指令中的敏感密碼。

## 🔍 利用方法
若系統管理不當，將使用者家目錄權限設為開放（如 `755`），普通使用者即可去讀取其他人的 `.bash_history` 檔案。我們可以執行以下指令，搜尋是否有人在下命令時明文留下了密碼：
```bash
find /home -type f -name .bash_history -exec cat {} \; 2>/dev/null
```
例如：在命令歷史中發現 `sshpass -p JZQuyIN5 ssh peter@localhost`，即可獲取 `peter` 的明文密碼。

---

# 實戰關聯
*   **靶機應用實例**：[[Stapler]]
