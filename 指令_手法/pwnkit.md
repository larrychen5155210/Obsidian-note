# Polkit 本地提權 (pwnkit)
本文件說明 Polkit 服務本地權限提升漏洞的使用方法。

## 💥 漏洞介紹
*   **CVE 編號**：CVE-2021-4034 (PwnKit)
*   **影響範圍**：Polkit 的 `pkexec` 命令列工具，幾乎所有主流 Linux 發行版在未安裝修補程式前皆受此漏洞影響。

## 🛠️ 利用方法
將編譯完成的 `PwnKit` 本地漏洞利用二進位檔上傳至目標靶機，執行以下指令完成秒破提權：
```bash
chmod +x PwnKit
./PwnKit
```
執行後即可直接在當前 Shell 取得 `root` 權限。

---

# 實戰關聯
*   **靶機應用實例**：[[Election1]]、[[Blogger]]
