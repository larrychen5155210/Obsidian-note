# 語言規則 (Language Rules)

- 請永遠使用繁體中文 (Traditional Chinese) 回答使用者的所有問題。

# 筆記寫作與架構規則 (Notebook Structure & Writing Rules)

## 1. 標籤 (Tags) 規範
- 所有的 Obsidian 標籤 (Tags) 均必須使用**全小寫**。
- 若標籤含有空格或特殊字元，請以連字號 `-` 或底線 `_` 替換以符合格式（例如：`serv-u-ftp-server`、`bash_history`、`writable-etc-passwd`）。
- **流程規範**：當使用者提供 writeup 連結要求整理時，代理應先自行判斷可以加入哪些 tags，並在正式寫入這些 tags 前，必須先詢問使用者的意見以獲得確認。

## 2. 指令與手法筆記組織
- 每一種滲透測試技術、工具或提權手法（如 `sqli`、`wordpress`、`mysql` , `cron-job`、`suid-serv-u-ftp-server`、`pwnkit`、`writable-etc-passwd` 等）必須建立為**獨立的筆記檔案**，並存放於 `指令/` 資料夾下，避免將多個不同手法混寫於單一檔案中。
- **重要限制：在建立手法筆記前，必須先詢問使用者的意見，確認哪些手法要建立成新筆記並加入指令資料夾。**

## 3. 雙向連結與錨點引用
- **簡介筆記 (Summary.md)** 與 **指令筆記 (Command.md)** 之間必須建立雙向連結：
  - 在簡介筆記中提到特定手法時，使用精準的章節錨點連結至指令筆記（例如：`[[mysql#💥 INTO OUTFILE 寫入 Web Shell|INTO OUTFILE]]`）。
  - 在指令筆記的末尾，必須建立 `# 實戰關聯` 段落並連結回對應的靶機簡介筆記（例如：`[[Stapler]]`）。
- **詳細筆記 (Walkthrough.md)** 應保持純淨，專注於紀錄靶機的真實滲透步驟與終端輸出結果，**不需**在正文中加入指向 `指令/` 資料夾的雙向連結。
