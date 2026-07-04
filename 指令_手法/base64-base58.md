# Base64 與 Base58 編碼解碼 (base64-base58)
本文件整理在滲透測試中常見的 Base64 與 Base58 編碼之識別與解碼手法。

## 🔍 Base64 編碼與解碼
Base64 編碼常用於在 HTTP 或設定檔中傳輸二進位數據。其特徵為字符集包含 `A-Z`、`a-z`、`0-9`、`+`、`/`，且結尾常用 `=` 進行填充。

### 🛠️ 終端機解碼方法
```bash
# 解碼 Base64 字串
echo -n "SGVsbG8gV29ybGQ=" | base64 -d

# 將檔案內容進行 Base64 解碼
base64 -d encoded_file.txt > decoded_output.txt
```

---

## 🔍 Base58 編碼與解碼
Base58 與 Base64 類似，但為了避免視覺混淆，移除了容易看錯的字符：`0` (零)、`O` (大寫 o)、`I` (大寫 i)、`l` (小寫 L)、`+` (加號) 和 `/` (斜線)。常用於區塊鏈（如比特幣位址）或特定 CTF 隱寫中。

### 🛠️ 終端機解碼方法
由於大部分 Linux 系統未內建 Base58 工具，可透過以下方式解碼：

#### 1. 使用 Python 的 `base58` 模組
若系統已安裝 Python 且有 `base58` 函式庫，可直接以命令列執行：
```bash
# 安裝庫 (如有需要)
pip3 install base58

# 解碼 Base58 字串
python3 -c "import base58; print(base58.b58decode('f1MgN9mTf9SNbzRygcU').decode('utf-8'))"
```

#### 2. 使用 apt 安裝工具 (Debian / Ubuntu)
```bash
# 安裝 base58 命令行工具
sudo apt install base58 -y

# 解碼 Base58 字串
echo -n "f1MgN9mTf9SNbzRygcU" | base58 -d
```

> [!tip] **線上工具解碼**
> 亦可將編碼字串貼至 **CyberChef**，搜尋並套用 `From Base58` 配方即可快速解碼。

---

# 實戰關聯
*   **靶機應用實例**：[[Gaara]]
