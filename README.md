# MC Discord Whitelist Sync(每一版將會多+1個readme)

透過 Discord 身分組驗證會員資格，讓玩家自助申請 Minecraft 伺服器白名單，不需要 RCON。

## 架構

```
玩家瀏覽器
   │ 1. 用 Discord 登入
   ▼
Google Apps Script 網頁 ──2. 用玩家自己的權杖查身分組──> Discord API
   │ 3. 有指定身分組才顯示表單
   │ 4. 玩家輸入 MC UUID 送出
   ▼
Google Sheet：Queue（待處理指令）+ Members（會員 ↔ MC 帳號對照）
   ▲ 5. 每分鐘輪詢
   │
你的伺服器上的 poller.js ──寫入 FIFO──> Minecraft 主控台
```

- 前端完全跑在 Google Apps Script（免費、有固定網址、不用自己架網站）
- 白名單資料進出走 Google Sheet 當佇列
- 你的主機只需要能連到外部網路（不需要固定 IP、不需要對外開 port）
- 不使用 RCON，改用 FIFO 模擬對 Minecraft 主控台輸入指令

## 為什麼不用 Bot Token 查身分組

一開始的實作是用 Discord Bot Token 查詢成員身分組，但實測發現從 Google Apps Script
呼叫這支 API 會被 Cloudflare 間歇性擋掉（回傳 `{"message":"internal network error","code":40333}`）。
改成用**玩家自己的 OAuth 權杖**（scope: `guilds.members.read`）去查自己在伺服器內的身分組，
繞開了這個問題。定期複查會員資格時，則用存下來的 `refresh_token` 重新換一組權杖來查，
不需要 Bot Token。

## 前置需求

- 一個 Google 帳號（跑 Apps Script + Google Sheets）
- 一個 Discord Application（[Discord Developer Portal](https://discord.com/developers/applications)）
- 一台可以跑 Node.js、且能連到你的 Minecraft 伺服器主控台的主機（VPS、家用主機都可以）
- Minecraft 伺服器（測試於 Paper，理論上任何看得懂 vanilla `whitelist add/remove` 指令的伺服器都適用）

## 部署步驟

### 1. 讓 Minecraft 伺服器接上 FIFO

FIFO（named pipe）是本專案不需要 RCON 的關鍵：把它接到 Minecraft 的 stdin，
之後寫文字進去就等於在主控台打指令按 Enter。

```bash
mkfifo /path/to/your/mc/mc-console.fifo
chmod 660 /path/to/your/mc/mc-console.fifo
```

用 `debian-poller/start-mc.sh` 啟動伺服器（先把裡面的 `SERVER_DIR`、`FIFO_PATH`、
記憶體參數改成你自己的），建議搭配 `tmux`（沒裝的話 `sudo apt install tmux -y`）：

```bash
chmod +x start-mc.sh
tmux new-session -d -s mc /path/to/start-mc.sh
```

測試 FIFO 有沒有接通：
```bash
echo 'say test from fifo' > /path/to/your/mc/mc-console.fifo
tmux attach -t mc   # 應該會看到 "test from fifo"
```

更完整的 systemd / screen 寫法可以參考 `debian-poller/setup-fifo.sh` 裡的說明。

### 2. 建立 Discord Application

1. 到 [Discord Developer Portal](https://discord.com/developers/applications) 建立一個新 Application
2. **OAuth2** 分頁：記下 `Client ID`、`Client Secret`
3. Redirect URI 先留空，等步驟 3 部署完 Apps Script 網頁後回來補上
4. 開發者模式（設定 → 進階）複製：伺服器 ID（Guild ID）、要拿來當「會員」判斷依據的身分組 ID

### 3. 建立 Google 試算表 + Apps Script

1. 建一個新的空白 Google 試算表
2. 「擴充功能 → Apps Script」，把 `apps-script/Code.gs` 貼進去覆蓋預設內容
3. 修改檔案最上面的常數：`DISCORD_CLIENT_ID`、`DISCORD_GUILD_ID`、`MEMBER_ROLE_IDS`
   （`MEMBER_ROLE_IDS` 是陣列，符合其中一個身分組就算會員，可以只放一個）
4. 新增一個 HTML 檔案，檔名 `index`，貼上 `apps-script/index.html` 的內容
5. 「專案設定 → 指令碼屬性」新增：
   - `DISCORD_CLIENT_SECRET`
   - `DISCORD_BOT_TOKEN`（只有每小時自動巡檢會用到，一般登入流程不需要）
6. 「部署 → 新增部署作業」→ 類型選「網頁應用程式」→ 執行身分：我 → 存取權限：**知道連結的使用者**
7. 部署完拿到 `.../exec` 網址，回 Discord Developer Portal 貼進 Redirect URI 存檔
8. Apps Script 編輯器上方函式下拉選單，依序執行一次：
   - `createHourlyTrigger`（每小時自動複查會員資格，過期自動退出白名單）
   - `createAccessNoersTrigger`（選用：見下方「無條件放行名單」）

之後要更新程式碼，用「管理部署作業 → 編輯 → 新版本」，不要建立全新的部署作業，
否則網址會變，Discord 的 Redirect URI 也要跟著改。

### 4. 部署 poller（你的主機上）

```bash
cd debian-poller
npm install
cp .env.example .env
# 編輯 .env：填入 SPREADSHEET_ID、服務帳戶金鑰路徑、FIFO 路徑、whitelist.json 路徑
node poller.js   # 先手動測試一次
```

**建立 Google 服務帳戶**（讓 poller.js 能讀寫試算表）：
1. [Google Cloud Console](https://console.cloud.google.com) 建立/選一個專案
2. 啟用 **Google Sheets API**
3. 「IAM 與管理 → 服務帳戶」建立服務帳戶 → 建立 JSON 金鑰並下載
4. 打開你的試算表 → 右上角「共用」→ 貼上服務帳戶 email → 權限選**編輯者**

**設定 cron 每分鐘跑一次**：
```bash
crontab -e
```
```
* * * * * cd /path/to/debian-poller && /usr/bin/node poller.js >> /path/to/debian-poller/poller.log 2>&1
```

## 資料流小抄

- `Members` 工作表：DiscordID / DiscordUsername / MCName / Status / JoinedAt / LastCheckedAt / MCUUID
- `RefreshTokens` 工作表：DiscordID / RefreshToken / UpdatedAt（自動建立，跟 Members 分開存）
- `Queue` 工作表：Timestamp / Action / Username / Status / ProcessedAt / Note
- `CurrentWhitelist` 工作表：poller.js 每次執行都會同步，鏡像目前 `whitelist.json` 的內容
- 玩家送出表單 → 寫入 Members（active）+ RefreshTokens + Queue（add, pending）
- 每小時巡檢 → 用該會員的 refresh token 重新查身分組，沒有的話從 Members 刪除、送出 Queue（remove, pending）
- poller.js 每分鐘處理 Queue 裡 pending 的列，執行完改成 done/error

## 選用功能

### 無條件放行名單（跳過 Discord 驗證）

想讓特定 Discord 帳號永遠不用驗證身分組就能用（例如管理員、VIP）：

1. 試算表新增分頁，命名 `discord_override`
2. A 欄填 Discord 使用者 ID，B 欄可寫備註

登入時如果 Discord ID 在這份名單裡，會直接跳過身分組檢查；巡檢時也不會被移除。

### 無條件放行名單（直接放行特定 MC 帳號，不綁定 Discord）

想讓特定 MC 帳號一直留在白名單，不透過網頁申請流程：

1. 試算表新增分頁，命名 `access_noers`
2. A 欄填 MC 使用者名稱

`ensureAccessNoers` 觸發條件（`createAccessNoersTrigger` 建立）每 10 分鐘會確保名單裡的人都在白名單上。

## 安全性提醒

- `DISCORD_CLIENT_SECRET`、`DISCORD_BOT_TOKEN`、服務帳戶金鑰都是敏感資訊，
  只放在 Script Properties／主機本機檔案，**絕對不要 commit 進版本控制**
  （本專案的 `.gitignore` 已經預先擋掉常見的檔名，但還是要自己檢查一次再 push）
- Bot 只需要「讀取伺服器成員」的基本權限，不需要任何管理權限
- refresh token 儲存在 Google Sheet 裡，等同帳號憑證，試算表的共用權限要設好，不要公開分享

## 授權

MIT License，詳見 [LICENSE](LICENSE)。
