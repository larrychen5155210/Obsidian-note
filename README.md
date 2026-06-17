# Obsidian 筆記同步指南 (GitHub)

這是一份用來引導如何將本地端的 Obsidian 筆記同步至 GitHub，以及如何在不同裝置間拉取筆記的指南

## 🛠️ 首次設定（將本地筆記連結至 GitHub） 

如果你本地端已經有 Obsidian 筆記，但尚未與 GitHub 儲存庫連結，請依序執行以下步驟：

1. **開啟 Terminal 並切換到你的 Obsidian 筆記資料夾路徑**

	```bash
	cd .\Desktop\Obsidian-note
	```

2. **初始化 Git 儲存庫**：

	```bash
	git init
	```

3. **排除不需要同步的檔案（重要）**：
在筆記根目錄下建立一個名為 `.gitignore` 的檔案，並寫入以下內容（避免將 Obsidian 的本地外觀設定或快取上傳）：
	```
	.obsidian/workspace
	...........
	```

4. **連結遠端 GitHub 儲存庫**（請將網址替換為你的 Repository 網址）：
	```bash
	git remote add origin https://github.com/larrychen5155210/Obsidian-note.git
	```

5. **將分支名稱改為 main**：
	```bash
	git branch -M main
	```

## ⬆️ 如何將本地端筆記「上傳」至 GitHub

當你在本地端新增或修改了筆記，想要同步到 GitHub 時，請執行以下指令：

1. **開啟 Terminal 並確保位於筆記資料夾中**

	```bash
	cd .\Desktop\Obsidian-note
	```

2. **將所有變更加入暫存區**：
	```bash
	git add .
	```

3. **提交變更並加上紀錄訊息**（雙引號內可自行修改）：
	```bash
	git commit -m "更新筆記內容"
	```

	**全新安裝的 Git 會遇到 :**
	```bash
	Run
	
	  git config --global user.email "you@example.com"
	  git config --global user.name "Your Name"
	```

4. **推送到 GitHub**：
	```bash
	git push -u origin main
	```
	(第一次加上 `-u`，之後只需輸入 `git push` 即可)

## ⬇️ 如何將 GitHub 上的筆記「拉回」本地端

如果你在其他裝置更新了筆記，或是直接在 GitHub 網頁上做了修改，想要把最新版本下載到目前的電腦：

**情況 A：這台電腦「已經有」這份 Git 專案**
1. 開啟終端機並切換到筆記資料夾
	```bash
	cd .\Desktop\Obsidian-note
	```

2. 執行拉取指令：
	```bash
	git pull origin main
	```

**情況 B：全新電腦，要「第一次下載」整份筆記**
1. 開啟終端機，切換到你想存放筆記的資料夾
	```bash
	cd .\Desktop
	```

2. 執行複製指令：
	```bash
	git clone https://github.com/larrychen5155210/Obsidian-note.git
	```

3. 下載完成後，直接用 Obsidian 軟體「開啟舊資料夾 (Open folder as vault)」，選擇該資料夾即可。

## 💡 自動化同步
如果你覺得每次都要輸入指令很麻煩，建議可以在 Obsidian 內安裝社群外掛：

- **外掛名稱**：`Obsidian Git`
- **功能**：它可以設定每隔幾分鐘自動執行 `git add / commit / push / pull`，讓你完全不用開終端機就能完成同步！