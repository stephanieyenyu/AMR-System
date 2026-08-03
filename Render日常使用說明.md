# Aurobox — 日常使用說明（系統已部署在 Render，不用安裝任何東西）

系統已經跑在 Render 上，24 小時開著，不依賴你自己的電腦。**任何電腦、平板、手機，只要有瀏覽器，就能直接使用**，不需要安裝 Docker、不需要跑任何 .bat 檔、不需要 ngrok。

## 使用網址

- **管理員後台（Dashboard）**：
  `https://你的line-backend網址.onrender.com/admin`

- **住戶不需要另外開網址**——住戶端全部透過 LINE 官方帳號操作，加好友、傳「門牌 姓名」綁定即可，到貨/抵達通知會直接推播到住戶的 LINE。

## 登入方式

打開 Dashboard 網址，瀏覽器會跳出帳號密碼輸入框，輸入你在 Render 環境變數裡設定的 `ADMIN_USERNAME` / `ADMIN_PASSWORD` 即可。

## 需要注意的地方

- **Render 免費/Starter 方案的服務，閒置一段時間沒人連線可能會進入休眠**，休眠後第一次打開網址會等個幾秒到十幾秒喚醒，屬正常現象，不是壞掉，重新整理或稍等一下就會正常顯示。
- 忘記帳密的話，去 Render Dashboard 的 `line-backend` 服務 → Environment 分頁，可以查看/修改 `ADMIN_USERNAME`、`ADMIN_PASSWORD` 的值。

---

> 如果之後需要**重新建立**一份全新的部署環境（例如要開第二套測試環境，或這套環境需要重建），才需要參考另一份《Render部署指南-從零開始.md》。日常使用不需要看那份。
