# 💥 Exiftool Metadata 敏感資訊洩漏與使用者列舉 (exiftool)

本文件說明如何利用 Exiftool 讀取文檔 (PDF、Office 文件等) 的元數據 (Metadata)，進而收集系統內部使用者帳號或軟體版本等敏感資訊。

---

## 🔍 漏洞成因與原理

文檔在創建或導出時，辦公軟體（如 MS Office、Adobe Acrobat）會自動在檔案內附加 **元數據 (Metadata)**。這些元數據通常包含了：
*   創建文檔的作者名稱 (Author)。
*   最後修改該文檔的使用者名稱。
*   文檔的創建時間與編輯時間。
*   創建時使用的軟體名稱與作業系統版本。

如果這些文檔被公開放置在網站或匿名 FTP 上，攻擊者下載後即可透過讀取元數據提取出有效的「人名/帳戶名」。這些收集到的帳號在 Active Directory 網域或外部登入點（如 SSH、Webmin、VPN）中，能直接用作 **使用者字典 (Username Wordlist)**，大幅提高密碼噴灑 (Password Spraying) 或暴力破解的成功率。

---

## 🛠️ 常見利用與繞過技巧

### 1. 資訊收集與單一提取
*   **讀取特定 PDF 的完整元數據**：
    ```bash
    exiftool NEWSLETTER-TEMPLATE.pdf
    ```
    *在輸出結果中尋找 `Author` 欄位（例如：`Author : Cassie`）。*

---

### 2. 批量提取技巧
若下載了大量的 PDF 或 Office 文檔，可以使用以下萬用字元指令進行批量作者欄位提取：

*   **批量提取當前目錄下所有 PDF 的作者**：
    ```bash
    exiftool -Author *.pdf
    ```
*   **批量提取並去重輸出為字典**：
    ```bash
    exiftool -T -Author *.pdf | sort -u > user_list.txt
    ```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[OSCP-C]]（在 Linux 外網靶機 192.168.x.157 上，攻擊者透過 anonymous 匿名登入 FTP，下載了 `backup` 目錄下的多個 PDF 範本。利用 `exiftool` 批量分析這些 PDF 的元數據，提取出多名文檔作者：`Mark`、`Cassie`、`Robert`，將其整理為用戶名單。最後成功以 `cassie:cassie` 弱密碼登入目標的 20000 Port Webmin 服務，取得初始存取點）。
