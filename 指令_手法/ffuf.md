# 💥 Web Fuzzing / 參數爆破 (fuzzing)

本文件說明使用 ffuf (Fuzz Faster U Fool) 進行 Web 目錄、檔案與參數爆破的檢測方法與常見利用技巧。

---

## 🔍 漏洞成因與原理

在滲透測試中，Web 應用程式常常會暴露出一些隱藏的目錄、敏感檔案或未公開的輸入參數（如本地檔案包含漏洞 LFI 中所使用的檔案包含參數）。為了識別這些隱藏的進入點，需要透過模糊測試 (Fuzzing) 發送大量的 HTTP 請求，並根據 Web 伺服器的回應狀態碼、回應大小 (Response Size) 等特徵來篩選出有效的參數或目錄。

> [!NOTE]
> `ffuf` 是使用 Go 語言開發的 Web Fuzzer，具有極高的併發效率與效能，是目前資安業界進行網頁目錄與參數爆破的主流工具。

---

## 🛠️ 常見利用與繞過技巧

### 1. 漏洞檢測 / 資訊收集

#### A. 參數爆破 (Parameter Fuzzing)
當懷疑某頁面存在未公開的參數（例如可能存在 LFI 漏洞的參數），且該頁面需要登入權限時，可以使用以下指令爆破參數名稱：
```bash
ffuf -u "http://<Target-IP>/path/to/page.php?FUZZ=../../../../etc/passwd" -w /usr/share/wordlists/dirb/common.txt -b "PHPSESSID=xxxxxxxx" -fs <預設回應大小>
```
*   `-u`：指定目標 URL，使用 `FUZZ` 作為參數字典的填充位置。
*   `-w`：載入常見參數名稱字典。
*   `-b`：傳入 Session Cookie 以繞過登入限制。
*   `-fs`：過濾並隱藏（Filter Size）特定大小的無效回應，只顯示有效結果。

#### B. 目錄/檔案爆破 (Directory/File Fuzzing)
爆破目標網站的隱藏目錄與指定副檔名的檔案，並過濾無效回應：
```bash
ffuf -u "http://<Target-IP>/FUZZ" -w /usr/share/wordlists/dirb/common.txt -e .php,.txt,.html -mc 200,301,302
```
*   `-e`：指定要爆破的檔案副檔名。
*   `-mc`：只匹配 (Match Code) 指定狀態碼的回應。

#### C. 使用 -fc 參數過濾特定狀態碼
除了匹配特定狀態碼，也可以使用 `-fc` 直接過濾並排除不需要的狀態碼回應：
```bash
ffuf -u "http://<Target-IP>/FUZZ" -w /usr/share/wordlists/dirb/common.txt -fc 404,403
```
*   `-fc`：過濾狀態碼 (Filter Code)。例如 `-fc 404,403` 可以隱藏狀態碼為 404 與 403 的無效或未授權網頁，只保留其他可能的狀態碼結果。

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[DC-9]]（利用 Union-based SQLi 取得管理員憑證並登入後台後，使用 `ffuf` 攜帶 Session Cookie 爆破 `/manage.php` 的 GET 參數，成功發現了 `?file` 參數，進而觸發 LFI 漏洞讀取 `/etc/knockd.conf` 以進行 Port Knocking 開啟 SSH）。
