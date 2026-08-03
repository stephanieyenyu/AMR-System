# Aurobox — Render 部署指南

> 這份文件記錄的是你目前實際在用的部署方式：**在 Render Dashboard 手動建立服務** 從一個全新、完全空白的 Render 帳號開始，一路走到整個系統能動。

---

## 0. 開始之前，你需要先準備好這些帳號/資料

- **GitHub**：這個 repo 的存取權限 → `github.com/stephanieyenyu/AMR-System`
- **Render 帳號**：[render.com](https://render.com) 免費註冊即可開始
- **LINE Developers 帳號**，並且已經建立：
  - 一個 **Messaging API Channel**（要拿 Channel Secret、Channel Access Token）
  - 一個 **LINE Login Channel**（要拿 Channel ID，兩個 LIFF App 要掛在這個 Channel 底下）
- **Pudu 開放平台**帳號/API Key（要接真實機器人才需要；沒有的話 `flashbot-robot` 那組環境變數先填假值，服務一樣能啟動，只是實際呼叫 Pudu API 會失敗）

---

## 1. 建立資料庫（Render Postgres）

1. Render Dashboard → **New** → **PostgreSQL**
2. 填寫：
   - Name：例如 `aurobox-postgres`
   - Region：選離你近的（例如 Singapore）
   - Plan：**Starter**（Free 方案的資料庫 90 天後會被自動刪除，正式使用不建議）
3. 建立完成後，進到這個資料庫的頁面，**Connect** 分頁裡有兩組連線字串：
   - **Internal Database URL**：只有你 Render 帳號底下的其他 Render 服務能連，速度快、不計流量費——`line-backend`、`flashbot-robot` 兩個服務都要用這組
   - **External Database URL**：從你自己電腦連（例如用 psql、DBeaver 檢查資料）才需要用這組
4. 兩個服務**可以共用同一個 Postgres 執行個體**（不用因為有兩個服務就開兩個資料庫），資料表本身不會互相打架（`line-backend` 用 `packages`、`line_binding` 這些表，`flashbot-robot` 用 `doors`、`robot_state`，表名不重複）

---

## 2. 建立 `line-backend` 服務

1. Render Dashboard → **New** → **Web Service**
2. 連接 GitHub，選 `stephanieyenyu/AMR-System` 這個 repo
3. 填寫：

   | 欄位 | 值 |
   |---|---|
   | Name | `line-backend` |
   | Region | 跟資料庫同一個 |
   | Root Directory | `line-backend` |
   | Runtime | Python 3 |
   | Build Command | `pip install -r requirements.txt` |
   | Start Command | `uvicorn app.main:app --host 0.0.0.0 --port $PORT` |
   | Plan | Starter |

4. **Environment** 分頁，逐一新增這些環境變數：

   | 變數 | 值 | 說明 |
   |---|---|---|
   | `LINE_CHANNEL_SECRET` | 從 LINE Developers Console 複製 | Messaging API Channel 的 Channel Secret |
   | `LINE_CHANNEL_ACCESS_TOKEN` | 從 LINE Developers Console 複製 | 要選「長期有效」那種 token |
   | `LIFF_ID` | 先留空 | 第 5 節建好 LIFF App 後再回來填 |
   | `LIFF_ID_RETURN` | 先留空 | 同上 |
   | `LINE_LOGIN_CHANNEL_ID` | 從 LINE Login Channel 複製 | 驗證 LIFF ID Token 用 |
   | `DATABASE_URL` | 見下方說明 | |
   | `ROBOT_API_BASE_URL` | 先留空 | 第 3 節建好 `flashbot-robot` 後回來填 |
   | `ROBOT_HOME_POINT_NAME` | `office` | 要跟 `flashbot-robot` 那邊的 `HOME_POINT_NAME` 填一樣的值 |
   | `ADMIN_USERNAME` | 自己取一個 | 不要用預設值 `aurotek` |
   | `ADMIN_PASSWORD` | 自己取一個夠複雜的 | 不要用預設值 `flashbot` |
   | `APP_ENV` | `production` | |

   **`DATABASE_URL` 特別注意**：把第 1 節複製到的 Internal Database URL 貼過來，但**開頭的 `postgresql://` 要手動改成 `postgresql+psycopg://`**（後面帳密/主機/資料庫名稱都不用動）——這個服務用的是 psycopg3 驅動，SQLAlchemy 靠這個前綴分辨要用哪個驅動。

5. 點 **Create Web Service**，等 build + 部署完成（第一次會比較久）。完成後，服務頁面最上方會顯示一個網址，長得像 `https://line-backend-xxxx.onrender.com`，記下來。

---

## 3. 建立 `flashbot-robot` 服務

1. Render Dashboard → **New** → **Web Service**（一樣連同一個 repo）
2. 填寫：

   | 欄位 | 值 |
   |---|---|
   | Name | `flashbot-robot` |
   | Region | 跟前面一樣 |
   | Root Directory | `flashbot-robot` |
   | Runtime | Python 3 |
   | Build Command | `pip install .` |
   | Start Command | `python run.py --host 0.0.0.0 --port $PORT` |
   | Plan | Starter |

   ⚠️ **兩個容易出錯的地方**：
   - Build Command 不是 `pip install -r requirements.txt`——這個資料夾**沒有** `requirements.txt`，是用 `pyproject.toml` 管理套件，要用 `pip install .`
   - Start Command **一定要帶 `--port $PORT`**——Render 會動態指定實際要監聽的 port，`run.py` 內建預設值是寫死的 `6000`，沒加這個參數，服務會顯示「啟動成功」但 Render 連不進去（外部一直是 502）

3. **Environment** 分頁：

   | 變數 | 值 | 說明 |
   |---|---|---|
   | `Pd_key` | Pudu 開放平台 API Key | |
   | `Pd_secret` | Pudu 開放平台 API Secret | |
   | `Aurotek_id` | Pudu 場域/Shop ID | |
   | `FLASHBOT_SN` | 機器人序號，例如 `8FF0559...` | |
   | `DEFAULT_MAP_NAME` | Pudu 後台的地圖名稱 | |
   | `HOME_POINT_NAME` | `office` | 要跟 `line-backend` 那邊的 `ROBOT_HOME_POINT_NAME` 一樣 |
   | `CHARGE_POINT_NAME` | 充電站點位名稱 | |
   | `DOOR_MODE` | `4_DOORS` 或 `3_DOORS` | |
   | `CENTRAL_API_BASE_URL` | 先留空 | 第 4 節填 |
   | `DATABASE_URL` | 見下方說明 | |

   ⚠️ **這幾個變數名稱不要打錯**：程式碼實際讀取的就是 `Pd_key`／`Pd_secret`／`Aurotek_id`／`FLASHBOT_SN` 這幾個名字（大小寫也要一致），不是 `APP_KEY`／`APP_SECRET` 這種看起來比較「標準」的名字——`render.yaml` 裡剛好就是打錯這幾個名字，如果之後要對照那份檔案要注意這個落差。

   `DATABASE_URL`：一樣貼第 1 節的 Internal Database URL，**這裡維持 `postgresql://` 開頭就好，不用加 `+psycopg`**（這個服務用的是 psycopg2）。

4. **Create Web Service**，等部署完成，記下這個服務的網址（`https://flashbot-robot-xxxx.onrender.com`）。

---

## 4. 把兩個服務串起來

1. 回到 `line-backend` 的 Environment，把 `ROBOT_API_BASE_URL` 填成 `flashbot-robot` 的網址
2. 回到 `flashbot-robot` 的 Environment，把 `CENTRAL_API_BASE_URL` 填成 `line-backend` 的網址
3. 存檔後兩邊都會自動重新部署一次

---

## 5. LINE Developers Console 設定

1. Messaging API Channel → **Messaging API** 分頁 → **Webhook settings**，Webhook URL 填：
   ```
   https://你的line-backend網址.onrender.com/webhook
   ```
   按 **Verify** 測試（要等服務部署完成才會成功），成功後打開 **Use webhook** 開關

2. 建立兩個 LIFF App（在 LINE Login Channel 底下的 **LIFF** 分頁）：
   - 第一個：Endpoint URL 填 `https://你的line-backend網址.onrender.com/liff/scan`，記下這個 LIFF ID
   - 第二個：Endpoint URL 填 `https://你的line-backend網址.onrender.com/liff/return-request`，記下這個 LIFF ID
   （LIFF App 跟 Endpoint URL 是一對一綁定，這兩個不能共用同一個）

3. 回到 `line-backend` 的 Environment，把 `LIFF_ID`、`LIFF_ID_RETURN` 填上剛剛這兩個 ID，存檔重新部署

> Render 給的網址是固定、永久有效的 `https://xxx.onrender.com`，不會像本機開發那樣每次重啟就換掉，**不需要另外弄 ngrok 固定網域**——這是用 Render 部署比本機測試省事的地方。

---

## 6. 第一次部署後：建立資料庫表格

`line-backend` 不會自動建表，需要手動跑一次 `python -m app.init_db`。因為 Starter 方案沒有 Shell、也沒有 Pre-Deploy Command，用這個方法：

1. 到 `line-backend` 的 **Settings**，把 Build Command 暫時改成：
   ```
   pip install -r requirements.txt && python -m app.init_db
   ```
2. **Save Changes**，會自動觸發一次部署，等 **Logs** 顯示建表完成
3. **改回原本的** `pip install -r requirements.txt`，再存檔一次（不改回來也不會壞，只是沒必要每次部署都重跑一次建表）

`flashbot-robot` 會在服務啟動時自動建表（`create_app()` 裡的 `db.create_all()`），不用手動處理這一步。

---

## 7. 完成，測試看看

1. 用 LINE 加官方帳號好友，傳「門牌 姓名」（例如 `3F-2 王小明`）完成綁定
2. 打開 `https://你的line-backend網址.onrender.com/admin`，用剛剛設定的 `ADMIN_USERNAME`/`ADMIN_PASSWORD` 登入
3. 建立一筆包裹，確認 LINE 有收到到貨通知

---

## 8. 之後要加新的資料庫欄位（跑一次性 migration script）時

流程跟第 6 節一樣：把 migration script 的執行指令暫時接在 Build Command 後面、部署一次、確認成功、再改回來。例如：

```
pip install -r requirements.txt && python migrate_add_某個欄位_column.py
```
