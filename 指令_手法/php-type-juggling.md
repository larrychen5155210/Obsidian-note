# 💥 PHP 弱類型比較與 strcmp 繞過 (php-type-juggling)

本文件說明 PHP 弱類型比較 (Type Juggling) 的安全漏洞，特別是當 `strcmp()` 函數的返回值與 `0` 進行鬆散比較 (`==`) 時的繞過手法。

---

## 🔍 漏洞成因與原理

在 PHP 中，`strcmp(string $str1, string $str2)` 函數用於比較兩個字串：
- 若 `$str1` 小於 `$str2`，返回 `< 0`。
- 若 `$str1` 大於 `$str2`，返回 `> 0`。
- 若兩字串相等，返回 `0`。

然而，當傳入的參數類型不是字串（例如將陣列傳入 `$str1` 或 `$str2`）時：
1. `strcmp()` 會無法處理而返回 `NULL`（在某些 PHP 版本中會同時觸發 Warning 警告，但不會中斷執行）。
2. 若程式碼使用鬆散比較（即 `==` 雙等號）來比對結果是否為 `0`：
   ```php
   if (strcmp($_POST['password'], $pass) == 0)
   ```
3. 在 PHP 的弱型別轉換中，`NULL == 0` 會被判定為 `true`。
4. 這將導致密碼驗證被成功繞過。

> [!NOTE]
> 在 PHP 8.0 以上版本中，`strcmp()` 對於非字串類型的處理已更為嚴格，此繞過方式在現代 PHP 環境中可能不再適用，但在舊版 PHP (PHP 5.x, 7.x) 中非常常見。

---

## 🛠️ 利用與繞過步驟

### 1. 攔截請求
使用代理工具（如 Burp Suite）攔截登入時的 POST 請求。

### 2. 修改參數為陣列
將原本的參數修改為陣列格式。例如：
*   **原始請求 payload**：
    ```http
    username=admin&password=yourpassword
    ```
*   **修改後繞過 payload**：
    ```http
    username=admin&password[]=
    ```
    或者：
    ```http
    username=admin&password[]=anything
    ```

後端 PHP 接收到該參數後，`$_POST['password']` 會被解析為一個陣列 `[]`，從而使 `strcmp()` 返回 `NULL`，通過 `== 0` 的驗證。

---

# 實戰關聯
*   **靶機應用實例**：[[Potato]]（在匿名 FTP 下載的 `index.php.bak` 備份檔中，發現管理員登入頁面使用 `strcmp($_POST['password'], $pass) == 0` 進行密碼驗證，透過將 `password` 改為陣列傳遞，成功繞過驗證登入後台）。
