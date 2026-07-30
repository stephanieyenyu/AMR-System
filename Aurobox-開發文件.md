# Aurobox 開發文件（Development Documentation）

> 本文件依據 `line-backend`（LINE / Dashboard 端）與 `flashbot-robot`（機器人 / Pudu API 端）兩個 Branch 的實際程式碼撰寫，目標是讓未來接手的開發者能快速理解架構、掌握兩端如何協作，並知道未來要修改功能時該從哪裡下手。
>
> 文中凡是程式碼未寫明、註解與實際邏輯有落差、或需要你確認實際部署方式的地方，一律標註 `[需補充]`。

---

## 1. 專案概述（Project Overview）

**Aurobox** 是一套社區型 AMR（Autonomous Mobile Robot，自動移動機器人）包裹配送系統。核心情境是：

- 郵差/管理員把包裹交給社區管理室，**管理員透過 Web Dashboard 登記包裹**。
- **住戶完全透過 LINE 官方帳號**（無需另外安裝 App）收到到貨通知、選擇取貨方式、掃碼開門、確認取貨/退貨。
- **機器人（Pudu Flashbot 硬體）** 負責實際的移動、艙門開關、螢幕顯示 QR Code 等物理動作，透過 Pudu 開放平台 API 被遠端控制。

整個系統由三個角色協作組成：

| 角色 | 對應 Repo/Branch | 職責 |
|---|---|---|
| LINE 後端 + 管理員 Dashboard | `line-backend`（`main` branch） | **唯一的業務邏輯與狀態真相來源**：包裹狀態機、LINE 對話、管理員操作介面、排程逾時處理 |
| 機器人硬體控制層 | `flashbot-robot`（`flashbot` branch） | 只負責「收指令、操作硬體」：艙門開關、呼叫 Pudu API 導航、機器人狀態查詢，**不重複維護業務狀態** |
| Pudu 開放平台 | 外部第三方服務 | 提供機器人導航、艙門控制、任務畫面顯示等底層硬體 API |

這個角色分工非常關鍵：**`line-backend` 是大腦，`flashbot-robot` 是手腳**。所有「這個包裹現在是什麼狀態、下一步該做什麼」的判斷都在 `line-backend`；`flashbot-robot` 收到指令就執行，執行完回報結果，本身不做業務判斷（例如它不知道「這是送貨還是退貨」背後的完整流程意義，只知道要開哪扇門、要不要顯示 QR）。

兩者之間**沒有共用資料庫**，也沒有 MQTT / WebSocket，純粹透過 **HTTP RESTful API（JSON）** 溝通（詳見第 2 章）。

---

## 2. 系統架構與資料流（System Architecture & Data Flow）

### 2.1 通訊協定總覽

```mermaid
flowchart LR
    subgraph LINE生態圈["LINE 生態圈"]
        User[住戶 LINE]
        LIFF[LIFF 頁面\n掃碼 / 退貨申請]
    end

    subgraph LineBackend["line-backend（FastAPI, Port 8000）"]
        Webhook["/webhook"]
        API["業務 API\n/packages/*, /door-tasks/*"]
        Dashboard["管理員 Dashboard\n(內嵌 HTML+JS)"]
        Scheduler["APScheduler\n逾時排程"]
        DB1[(PostgreSQL\naurobox_line)]
    end

    subgraph RobotModule["flashbot-robot（Flask, Port 5000/6000）"]
        RobotAPI["硬體控制 API\n/api/door-tasks/*, /api/robot/*"]
        BgTasks["背景執行緒\n輪詢 / 排隊開門"]
        DB2[(PostgreSQL\naurobox_db)]
    end

    subgraph Pudu["Pudu 開放平台（外部）"]
        PuduAPI["custom_call2 / control_doors /\ncustom_content / status..."]
    end

    Admin[管理員瀏覽器] -->|HTTP Basic Auth| Dashboard
    User -->|Message/Postback| Webhook
    User -->|掃碼/申請退貨| LIFF
    LIFF --> API
    Webhook --> API
    Dashboard --> API
    API <--> DB1
    Scheduler --> API

    API -->|"call_robot_api()\nPOST/GET"| RobotAPI
    RobotAPI -->|"/door-tasks/{id}/arrived\n(唯一的回呼)"| API
    RobotAPI <--> DB2
    RobotAPI --> BgTasks
    BgTasks --> PuduAPI
    RobotAPI --> PuduAPI
    BgTasks -.轮询.-> PuduAPI
```

重點觀察：

1. **`line-backend` → `flashbot-robot`**：單向為主。`line-backend` 呼叫機器人動作 API 時，全部走 `app/main.py` 裡的 `call_robot_api()` 這個統一封裝（含 timeout、`retries=1` 重試、失敗寫入 `TaskLog`）。目標網址由環境變數 `ROBOT_API_BASE_URL` 決定。
2. **`flashbot-robot` → `line-backend` 的回呼（callback）只有一個**：機器人抵達住戶點位後，由 `src/aurobox/tasks.py` 的 `_poll_notify_display_qr()` 背景執行緒主動 POST 到 `{CENTRAL_API_BASE_URL}/door-tasks/{door_task_id}/arrived`。這是整個系統中唯一一個「機器人端主動通知中央大腦」的入口。
3. 其餘所有「機器人有沒有回到管理室」之類的狀態確認，**是 `line-backend` 主動輪詢**機器人端的 `GET /api/dashboard/status`（見 `poll_robot_returned()` 排程，每 20 秒一次），不是機器人推播回來的。`[需補充]`：`line-backend/app/main.py` 中還存在一支 `POST /packages/{package_id}/returned` 端點，但在目前 `flashbot-robot` 程式碼中找不到任何呼叫它的地方；研判是門任務改用 `door_task_id` 分組之前的舊端點，實際返航偵測已改由 `poll_robot_returned()` 輪詢取代。建議確認此端點是否已是死碼，可考慮移除或註記為 deprecated。
4. **`flashbot-robot` → Pudu 開放平台**：所有跟硬體有關的動作（導航、開關門、畫面切換）最終都會透過 `src/aurobox/pudu_client.py` 呼叫 Pudu 的 REST API，並用 HMAC-SHA1 簽章驗證身份（見 3-2 節）。
5. 兩邊資料庫完全獨立：`line-backend` 用 `aurobox_line`（`psycopg`／psycopg3 驅動），`flashbot-robot` 用 `aurobox_db`（`psycopg2` 驅動，或退化用本機 SQLite，見 4.2 節）。**沒有任何跨庫查詢**，唯一的溝通管道是 HTTP。

### 2.2 完整資料流程：從「住戶在 LINE 下指令」到「機器人動作」

以最常見的「送貨」完整流程為例（多包裹/多門情境用 `door_task_id` 分組，細節見 3-1 節）：

```mermaid
sequenceDiagram
    participant Admin as 管理員 Dashboard
    participant LB as line-backend
    participant LineApp as 住戶 LINE
    participant RB as flashbot-robot
    participant Pudu as Pudu 平台

    Admin->>LB: POST /packages（建立包裹）
    LB->>LineApp: push Flex Message「取貨/預約取貨/不收」
    LineApp->>LB: Postback PICKUP_NOW
    LB->>LB: status → pickup_now
    Admin->>LB: POST /packages/{id}/place（選艙門）
    LB->>RB: POST /api/door-tasks/{id}/assign
    RB->>Pudu: custom_call2（導航回管理室）
    RB-->>RB: 背景執行緒等待抵達 → 開門(LOADING)
    Admin->>RB: (管理員實際放包裹)
    Admin->>LB: POST /admin/dispatch-batch
    LB->>RB: POST /api/doors/load（關門→FULL）
    LB->>RB: POST /api/robot/dispatch（door_task_id+unit）
    RB->>Pudu: custom_call2（QR_CODE 模式，前往住戶點位）
    RB-->>RB: 背景輪詢 wait_until_arrived()
    RB->>LB: POST /door-tasks/{id}/arrived（唯一回呼）
    RB->>Pudu: custom_content（螢幕顯示 QR）
    LB->>LineApp: push「已抵達，請掃碼」+ LIFF 連結
    LineApp->>LB: LIFF 掃碼 → POST /door-tasks/{id}/pickup-complete
    LB->>LB: 驗證 LIFF ID Token + 掃描內容 == door_task_id
    LB->>RB: POST /api/door-tasks/{id}/pickup-complete
    RB->>Pudu: custom_complete（收掉任務畫面）+ control_doors（開門, PICKING）
    LineApp->>LB: LIFF按「取貨完成」→ POST /door-tasks/{id}/complete
    LB->>RB: POST /api/door-tasks/{id}/complete
    RB->>Pudu: control_doors（關門, 釋放為 EMPTY）
    RB-->>RB: 若全空 → 自動 custom_call2 返回管理室
    LB->>LB: advance_trip_or_return()：檢查下一站/是否要主動叫返航
```

其他重要分支流程（均已在程式碼中實作）：

- **拒收 / 逾時未取**：住戶按「拒收」或 8 分鐘未動作 → `handle_reject_at_door()` / `check_pickup_timeout()` → 狀態轉 `rejected_at_door` / `returned_timeout` → 呼叫機器人 `POST /api/door-tasks/{id}/cancel`（關門+收任務畫面，包裹仍在艙內帶回管理室）→ `advance_trip_or_return()` 決定下一站或返航。
- **不收（到貨通知當下直接拒絕）**：`status = voided`，包裹從沒出過管理室，不涉及機器人動作。
- **退貨**：住戶透過 LIFF（`/liff/return-request`）建立 `task_type=return` 包裹，一開始就是 `pickup_now`，走跟送貨幾乎相同的艙門分配/派送/掃碼流程，只是方向相反（機器人帶著空艙門去收件，帶回管理室後管理員需「確認取出」）。
- **緊急召回（Recall）**：管理員在 Dashboard 按下「返回管理室」→ `POST /admin/robot/recall` → `line-backend` 呼叫 `POST /api/robot/recall`；`flashbot-robot` 端會判斷機器人是否正在服務住戶（`PICKING`/`PUTTING`/剛抵達等），若是則**排入佇列等流程結束再召回**（`_wait_and_execute_recall`），否則立即取消任務並導航回管理室，同時把所有進行中艙門保護成 `FULL`。
- **逾時排程總覽**（皆在 `line-backend/app/main.py` 由 APScheduler 驅動）：

  | 排程函式 | 週期 | 用途 |
  |---|---|---|
  | `check_pickup_timeout` | 1 分鐘 | 送貨 `arrived` 超過 8 分鐘未取貨 → 自動觸發拒收流程 |
  | `check_assign_timeout` | 1 分鐘 | 艙門分配了但管理員一直沒裝載 → 逾時釋放艙門 |
  | `check_return_timeout` | 1 分鐘 | 退貨/拒收開門檢查超過 8 分鐘沒關門 → 強制關門 |
  | `poll_robot_returned` | 20 秒 | 輪詢機器人是否已回到管理室，補上 `returned_at` |
  | `check_stuck_dispatch` | 2 分鐘 | 安全網：確保「派送下一站失敗」不會讓整個批次卡死 |

  這幾支逾時邏輯都設在「8 分鐘」，是刻意小於 Pudu 機器人「超過 10 分鐘無動作會進入異常」的硬體限制，兩端開發者若要調整逾時時間，務必同時考慮這個硬體上限。

---

## 3. 模組開發說明（Module Walkthrough）

### 3-1 LINE & Dashboard 端（`line-backend`，`main` branch）

**框架與套件**（見 `line-backend/requirements.txt`）：

- Web 框架：`FastAPI` + `Uvicorn`（ASGI）
- ORM／資料庫：`SQLAlchemy 2.x`（同步 `Session`，非 async ORM）+ `PostgreSQL`（`psycopg[binary]` / `psycopg2-binary`）
- LINE 整合：`line-bot-sdk>=3.19.0`（v3 API：`WebhookParser`、`MessagingApi`、`FlexMessage`、LIFF）
- 排程：`APScheduler`（`BackgroundScheduler`，常駐在同一個 process 內，非獨立 worker）
- 設定讀取：`pydantic-settings`（`.env` 檔案）

**管理員 Dashboard 的實作方式（重要）**：Dashboard **不是**獨立的前端專案、沒有 build 流程，而是**原生 HTML + Vanilla JS 字串，直接內嵌在 `app/main.py` 裡**（`ADMIN_DASHBOARD_HTML`、`ADMIN_REPORTS_HTML`、`ADMIN_EXCEPTIONS_HTML`、`ADMIN_RESIDENTS_HTML` 這幾個 Python 常數），由 FastAPI 以 `HTMLResponse` 直接回傳。前端透過 `fetch()` 呼叫同一個 FastAPI app 裡的 JSON API。

> ⚠️ **已知踩雷紀錄**：這些 HTML 字串是 Python 的 triple-quoted string，內嵌的 JavaScript 若要換行字元，Python 會把 `\n` 直接吃成換行；JS 字面上要表示換行必須寫成 `\\n`（雙反斜線），否則字串會在錯誤位置斷開，曾造成正式環境白屏。修改任何 `*_HTML` 常數時務必留意這點。

**關鍵檔案位置**：

| 檔案 | 內容 |
|---|---|
| `app/main.py` | **所有東西的入口**：FastAPI app、LINE Webhook 解析與事件分派（`handle_postback`、`handle_text_binding`…）、全部業務 API 路由、管理員 Dashboard HTML/JS 字串常數、APScheduler 排程任務定義與啟動 |
| `app/models.py` | SQLAlchemy ORM：`Package`（核心狀態機）、`LineBinding`（門牌綁定）、`PackageRecipient`（一筆包裹的所有收件人）、`TaskLog`（事件稽核紀錄） |
| `app/line_messaging.py` | 封裝所有 LINE Messaging API 呼叫：到貨通知 Flex Message（`push_arrival_notification`）、抵達通知（`push_arrived_notification`）、純文字推播/回覆 |
| `app/line_verify.py` | 驗證 LIFF 傳來的 ID Token（呼叫 LINE 官方 `oauth2/v2.1/verify`），回傳 claims（`sub` 即為 LINE User ID） |
| `app/config.py` | 環境變數設定（`pydantic_settings.BaseSettings`），單例 `settings` |
| `app/db.py` | SQLAlchemy engine / `SessionLocal` / `get_db()` dependency |
| `app/init_db.py` | 一次性腳本：依 `models.py` 建立資料表（`python -m app.init_db`） |

**核心資料模型 `Package` 狀態機**：

```
pending → pickup_now → delivering → arrived → completed
                                        ├──▶ rejected_at_door（拒收 / 退貨取消）
                                        └──▶ returned_timeout（逾時未取）
pending → voided（到貨當下直接不收）
```

**`door_task_id` 是理解整個後端的關鍵概念**：同一位收件人、同一次派送，如果用到不只一扇門，這些包裹會共用同一組 `door_task_id`（UUID）。所有機器人回呼／掃碼／完成／拒收，都是以 `door_task_id` 為單位一次處理整組（見 `main.py` 內大量 `Package.door_task_id == task_uuid` 的查詢）。**同一戶如果同時有送貨與退貨兩個任務，會是兩個獨立的 `door_task_id`**（`task_type` 不同），彼此不共用艙門、也不假設對方進度（`is_door_actively_held()` 函式即是這個判斷的共用邏輯）。

處理 LINE Webhook／前端面板資料的關鍵函式：

- `POST /webhook`：解析 LINE 事件（`FollowEvent`/`UnfollowEvent`/`PostbackEvent`/`MessageEvent`），逐一事件包 `try/except`，避免一個事件失敗拖垮整批。
- `handle_postback()`：依 `action=` 分派（`PICKUP_NOW`／`SCHEDULE_PICKUP`／`REJECT`／`PICKUP_DONE`／`REJECT_AT_DOOR`／`CANCEL_RETURN`）。
- 管理員操作 API 都掛 `dependencies=[Depends(require_admin_auth)]`（HTTP Basic Auth），例外的是 `/webhook`、`/liff/*`、`/door-tasks/{id}/arrived`、`/packages/{id}/returned`（機器人／LINE 官方會呼叫，沒有帳密）。

### 3-2 機器人與 Pudu API 端（`flashbot-robot`，`flashbot` branch）

**框架與套件**（見 `flashbot-robot/pyproject.toml`）：

- Web 框架：`Flask` + `Flask-SQLAlchemy`
- 資料庫：`PostgreSQL`（`psycopg2-binary`），或無設定 `DATABASE_URL` 時退化為本機 SQLite（`instance/aurobox.db`，見 `src/aurobox/app.py` 的 fallback 邏輯）
- 與 Pudu 的簽章：`hashlib`/`hmac`（HMAC-SHA1，見 `pudu_client.py` 的 `PuduAuth`）

**如何與 Pudu 機器人建立連線**：`src/aurobox/pudu_client.py` 的 `PuduApiClient` 是唯一直接呼叫 Pudu Open Platform（`https://css-open-platform.pudutech.com`）的地方。每個請求都會：

1. 用 `x-date`（UTC 時間，RFC 格式）與（POST 時）`Content-MD5` 組出簽章字串。
2. 用 `app_secret`（`Pd_secret`）以 HMAC-SHA1 簽章，包在 `Authorization: hmac id="...", algorithm="hmac-sha1", headers="x-date", signature="..."` 標頭中送出。
3. `app_key`（`Pd_key`）與 `shop_id`（`Aurotek_id`）用來識別呼叫方與場域。

呼叫的核心 Pudu API（皆封裝在 `pudu_client.py`，`robot.py` 的 `FlashbotController` 再包一層業務語意）：

| Pudu API | 封裝方法 | 用途 |
|---|---|---|
| `GET v1/status/get_by_sn`、`GET v2/status/get_by_sn` | `get_status_v1` / `get_by_sn2` | 機器人即時狀態（合併 v1/v2 兩個來源） |
| `GET v1/robot/task/state/get` | `get_task_state` | 任務狀態（語意層面，不當作最即時的移動訊號） |
| `GET v1/robot/get_position` | `get_position` | 座標位置 |
| `GET map-service/v1/open/list`、`GET map-service/v1/open/map` | `get_map_list` / `open_map` | 地圖清單 / 開啟地圖 |
| `GET v2/recharge` | `recharge` | 回充電站（僅在艙門皆空時允許，見 `api.py` 的 `robot_recharge`） |
| `POST v1/custom_call` | `custom_call` / `custom_call2` | **導航指令**：叫機器人前往某個 `point`（管理室 / 住戶點位 / 充電站），可帶 `call_mode='QR_CODE'` |
| `POST v1/custom_content` | `custom_content` | 切換機器人螢幕畫面（例如顯示取件 QR Code） |
| `POST v1/custom_call/complete` | `custom_complete` | 收掉/清除目前任務畫面 |
| `POST v1/custom_call/cancel` | `custom_call_cancel` | 中斷目前任務（緊急召回、逾時取消時使用） |
| `POST v1/control_doors` | `control_doors` | 批次開關艙門（`FlashbotController.control_doors` 內建 3 門/4 門邏輯轉物理門對應） |
| `GET v1/door_state` | `get_door_state` | 查詢硬體回報的門狀態（CLI 用，主流程走本地 DB 的 `Door` 表為準） |

`FlashbotController.get_status_summary()`（`robot.py`）是很重要的一層抽象：它同時打 v1/v2/task-state 三個來源、各自 `try/except` 容錯，合併出穩定欄位（`state`/`move_state`/`run_state`/`is_charging`/`battery_level`/`current_location`），**所有上層程式碼（含輪詢、Dashboard 狀態顯示）都吃這個合併後的結果**，不直接讀某一個原始來源，未來 Pudu API 若有變動只需要改這一處。

**負責硬體控制指令與狀態回報的關鍵檔案**：

| 檔案 | 內容 |
|---|---|
| `src/aurobox/app.py` | Flask app factory（`create_app`）：資料庫初始化、艙門預設值重置（開機時強制關閉所有實體門、狀態歸零）、Blueprint 註冊 |
| `src/aurobox/api.py` | 所有 HTTP 路由（`api_bp` Blueprint）：`/door-tasks/{id}/assign`、`/doors/load`、`/robot/dispatch`、`/door-tasks/{id}/pickup-complete`、`/door-tasks/{id}/complete`、`/door-tasks/{id}/cancel`、`/door-tasks/return`、`/doors/return-open` / `return-complete` / `return-timeout`、`/dashboard/status`、`/robot/recall`、`/robot/recharge` |
| `src/aurobox/robot.py` | `FlashbotController`：業務語意層，包住 `PuduApiClient`，提供 `wait_until_arrived()`（輪詢直到抵達）、`get_status_summary()`（多來源狀態合併）、`control_doors()`（邏輯門→實體門映射） |
| `src/aurobox/pudu_client.py` | `PuduApiClient` + `PuduAuth`：Pudu API 的低階 HTTP 呼叫與 HMAC 簽章，並把每一次指令/回應寫入 `instance/robot_commands.log`（見 `_create_robot_command_logger`） |
| `src/aurobox/services.py` | `update_robot_state()`（寫入 `RobotState` 表）、`check_and_return_home_if_empty()`（艙門皆空時自動觸發返航） |
| `src/aurobox/tasks.py` | 背景執行緒：`_queue_door_action`（統籌開/關門順序，避免同時操作硬體衝突）、`_poll_notify_display_qr`（輪詢抵達 → 回呼中央大腦 → 顯示 QR）、`_wait_and_execute_recall`（排隊召回） |
| `src/aurobox/models.py` | `Door`（`sn`+`door_number` 唯一、狀態 enum：`empty/assigned/loading/full/picking/putting`）、`RobotState`（`sn` 唯一、`last_point`、`current_task_id`） |
| `src/aurobox/utils.py` | `build_custom_call_payload()`：組出 Pudu `custom_call2` 標準 payload |
| `src/aurobox/config.py` | 讀取 Pudu / DB 相關環境變數 |
| `src/aurobox/cli.py` | 命令列工具，方便手動測試單一 Pudu API（`aurobox --sn ... status` 等） |
| `run.py` | 程式進入點（`python -u run.py --debug`） |

**3 門 / 4 門相容設計**：`DOOR_MODE=3_DOORS` 時，邏輯上的 `H_01` 會同時對應實體的 `H_01`+`H_02` 兩扇門（見 `FlashbotController.control_doors` 裡的映射邏輯），對上層 `line-backend` 完全透明——`line-backend` 永遠只看到邏輯門號，不需要知道現場是 3 門還是 4 門櫃體。

---

## 4. Docker 部署與環境設定（Docker Deployment & Environment）

### 4.1 兩個服務的 Dockerfile 現況

| 服務 | Dockerfile 位置 | Base Image | 對外埠 | 啟動指令 |
|---|---|---|---|---|
| `line-backend` | `line-backend/Dockerfile` | `python:3.9-slim` | `8000` | `uvicorn app.main:app --host 0.0.0.0 --port 8000` |
| `flashbot-robot` | `flashbot-robot/Dockerfile` | `python:3.10-slim` | `EXPOSE 5000` | `python -u run.py` |

> ⚠️ **`[需補充]` 發現的埠號不一致**：`flashbot-robot/run.py` 的 `--port` 參數預設值是 **6000**，但 Dockerfile 只 `EXPOSE 5000` 且 `CMD` 沒有帶 `--port` 參數，README 又寫「預設服務位置：`http://0.0.0.0:5000`」。三者互相矛盾，容器實際上會監聽在 6000，但只對外曝露 5000 埠。**部署前請先確認實際要用哪個埠**，並二擇一修正：（a）在 Dockerfile 的 `CMD` 加上 `--port 5000`，或（b）把 `run.py` 的預設值改成 5000、`EXPOSE` 改成 6000。

### 4.2 環境變數

**`line-backend/.env`**（範本見 `line-backend/.env.example`）：

| 變數 | 說明 | 備註 |
|---|---|---|
| `LINE_CHANNEL_SECRET` | LINE Messaging API Channel Secret | 用於 Webhook 簽章驗證 |
| `LINE_CHANNEL_ACCESS_TOKEN` | LINE Messaging API Channel Access Token | 用於推播/回覆訊息 |
| `LINE_LOGIN_CHANNEL_ID` | LINE Login Channel ID | 用於驗證 LIFF 傳來的 ID Token（`verify_liff_id_token`），須與建立 LIFF App 用的 Channel 一致 |
| `LIFF_ID` | 掃碼取貨用 LIFF App ID | Endpoint URL 需設為 `/liff/scan` |
| `LIFF_ID_RETURN` | 退貨申請用 LIFF App ID | Endpoint URL 需設為 `/liff/return-request`；**必須是另一個獨立的 LIFF App**（LIFF App 與 Endpoint URL 是一對一綁定） |
| `DATABASE_URL` | PostgreSQL 連線字串 | 格式 `postgresql+psycopg://user:pass@host:5432/aurobox_line`（注意是 `+psycopg`，psycopg3） |
| `ROBOT_API_BASE_URL` | `flashbot-robot` 服務的對外網址 | 開發期常設成 ngrok URL，正式環境應改為內網服務名稱或穩定網址 |
| `APP_ENV` | `development` / `production` | `[需補充]`：程式碼中目前只在健康檢查回應中顯示，尚未看到依此切換任何行為（例如 debug、CORS），若有計畫依環境切換行為請一併補充 |
| `ADMIN_USERNAME` / `ADMIN_PASSWORD` | Dashboard HTTP Basic Auth 帳密 | **`config.py` 內建預設值 `aurotek` / `flashbot`，正式環境務必在 `.env` 覆寫**，否則等同無驗證 |
| `ROBOT_HOME_POINT_NAME` | 機器人「在管理室」時 `current_location` 應等於的字串 | 用於「機器人是否在管理室，才允許手動開關門」的判斷（`require_robot_at_office`），須與 `flashbot-robot` 端 `HOME_POINT_NAME` 一致 |

**`flashbot-robot/.env`**（範本見 `flashbot-robot/.env.example`）：

| 變數 | 說明 | 備註 |
|---|---|---|
| `Pd_key` / `Pd_secret` | Pudu 開放平台 API Key / Secret | 對應 `pudu_client.py` 的 HMAC 簽章 |
| `Aurotek_id` | Shop ID | Pudu 場域識別碼 |
| `FLASHBOT_SN` | 機器人序號 | `.env.example` 內附帶一組實機序號 `8FF055923050007`，**這是特定裝置的序號，換機器人／換場域必須更新** |
| `DEFAULT_MAP_NAME` | 預設地圖名稱 | 需與 Pudu 後台地圖名稱一致 |
| `HOME_POINT_NAME` | 管理室點位名稱 | 須與 `line-backend` 端 `ROBOT_HOME_POINT_NAME` 一致 |
| `CHARGE_POINT_NAME` | 充電站點位名稱 | |
| `DOOR_MODE` | `4_DOORS` / `3_DOORS` | 決定邏輯門↔實體門映射 |
| `CENTRAL_API_BASE_URL` | `line-backend` 服務的對外網址 | 機器人抵達回呼（`/door-tasks/{id}/arrived`）會打這個網址，開發期常設成 ngrok URL |
| `DATABASE_URL` | PostgreSQL 連線字串 | 格式 `postgresql://user:pass@host:5432/aurobox_db`（psycopg2，注意跟 `line-backend` 用的驅動前綴不同，且是**完全不同的資料庫**） |

`[需補充]`：`flashbot-robot/.env.example` 裡還列了 `LINE_CHANNEL_ACCESS_TOKEN`、`LINE_CHANNEL_SECRET`、`FLASK_ENV`、`FLASK_DEBUG`、`SERVER_HOST`、`SERVER_PORT`，但目前 `src/aurobox/config.py` 的 `load_config()` 並未讀取這幾個變數（Flask 的 host/port 實際上是由 `run.py` 的 `argparse` 決定，不是讀環境變數）。建議確認這些欄位是否為舊版殘留、之後計畫使用、或應從 `.env.example` 移除，避免造成維護者誤解。

### 4.3 建置與啟動（無 docker-compose 版）

目前 Repo 內**沒有附上 `docker-compose.yml`**，以下為兩個容器各自建置/啟動的基本指令；如果你已經在正式環境用自己的 `docker-compose.yml` 或其他編排工具部署，請忽略以下、直接補上你實際使用的設定 `[需補充：實際 docker-compose / K8s manifest]`。

```bash
# 1. 建置兩個 image
docker build -t aurobox-line-backend ./line-backend
docker build -t aurobox-flashbot-robot ./flashbot-robot

# 2. 準備一個共用 network，讓兩個容器能用容器名稱互相連線
docker network create aurobox-net

# 3. 啟動 line-backend（記得掛載 .env）
docker run -d --name aurobox-line-backend \
  --network aurobox-net \
  -p 8000:8000 \
  --env-file ./line-backend/.env \
  aurobox-line-backend

# 4. 啟動 flashbot-robot（記得掛載 .env，並掛 volume 保留 instance/ 目錄）
docker run -d --name aurobox-flashbot-robot \
  --network aurobox-net \
  -p 5000:6000 \
  --env-file ./flashbot-robot/.env \
  -v aurobox-robot-instance:/app/instance \
  aurobox-flashbot-robot
```

> 若兩容器在同一個 Docker network 下部署，可以把 `.env` 裡的 `ROBOT_API_BASE_URL` / `CENTRAL_API_BASE_URL` 直接改成容器名稱（例如 `http://aurobox-flashbot-robot:6000`、`http://aurobox-line-backend:8000`），**不需要繼續用 ngrok**——ngrok 目前是「開發期兩台機器/跨網路測試」用的臨時方案（見 `start-ngrok.bat` 與兩邊 `.env.example` 內的 ngrok 網址），正式部署建議改走內網或反向代理 + TLS。`[需補充]`：實際正式環境的網路拓樸（VPN／固定 IP／內網 DNS／TLS 憑證）尚未在程式碼中定案，需另行規劃。

**資料庫**：兩個服務各自需要一個 PostgreSQL 資料庫（`aurobox_line`、`aurobox_db`），可以是同一台 PostgreSQL 主機下的兩個 database，也可以是各自獨立的實例。首次啟動：

```bash
# line-backend：建立資料表（連到正確的 DATABASE_URL 後）
python -m app.init_db

# flashbot-robot：db.create_all() 已內建在 create_app() 裡，容器啟動時會自動建表 + 重置艙門狀態
```

### 4.4 檢視服務運行 Log

- **一般應用程式 Log**：`docker logs -f aurobox-line-backend`、`docker logs -f aurobox-flashbot-robot`（Uvicorn / Flask 的 stdout，含 SQLAlchemy 的 SQL echo——`app/db.py` 目前 `create_engine(..., echo=True)`，正式環境流量大時建議評估是否關閉以減少雜訊 `[需補充]`）。
- **機器人硬體指令專屬 Log**：`flashbot-robot` 另外有一份獨立的 rotating log，記錄每一次呼叫 Pudu API 的指令與回應（`instance/robot_commands.log`，由 `src/aurobox/pudu_client.py` 的 `_create_robot_command_logger` 建立，最大 1MB、保留 3 份備份）。這份 log 對排查「機器人到底收到什麼指令、Pudu 回了什麼」非常關鍵，**容器化部署務必用 volume 掛載 `instance/` 目錄**，否則容器重啟就會遺失（`instance/` 目前在 `.gitignore` 中不會進版控，也不會隨 image 保留）。
- **每日業務事件 Log（非硬體層）**：`line-backend` 的 `TaskLog` 資料表（見 `app/models.py`）記錄所有業務事件（建立/派送/抵達/完成/拒收/逾時…），可透過 Dashboard 的「每日報表」頁面（`/admin/reports`）查詢，比翻 stdout log 更適合追蹤特定包裹的完整生命週期。

---

## 5. 未來開發與除錯指南（Future Development & Troubleshooting）

### 5.1 新增一個「機器人動作」該改哪裡

假設要新增一個全新的硬體動作（例如「單獨開啟指定一扇門讓管理員檢查」），建議順序：

1. **`flashbot-robot/src/aurobox/pudu_client.py`**：如果 Pudu 那邊本來就有對應的 API，在這裡新增一個低階呼叫方法（依照現有 `control_doors`／`custom_call2` 的寫法：組 payload → `self._post()` → log）。
2. **`flashbot-robot/src/aurobox/robot.py`**：在 `FlashbotController` 加一個業務語意方法包住上一步（例如處理 3 門/4 門映射、預設參數）。
3. **`flashbot-robot/src/aurobox/api.py`**：新增一個 Flask route，內部呼叫 `current_app.pudu_controller` 上一步的方法，並更新本地 `Door` / `RobotState` 資料表反映結果。若牽涉到「需要等機器人抵達才能動作」，仿照 `tasks.py` 裡 `_queue_door_action` 的模式，用 `threading.Thread` 起背景執行緒、透過全域 `Lock`/`Queue` 避免同時操作硬體衝突。
4. 如果新增了狀態或欄位，記得同步更新 `flashbot-robot/src/aurobox/models.py`（`Door` 的 `CheckConstraint` 目前寫死允許的門號與狀態值，新增狀態要記得加進 constraint）。
5. **`line-backend/app/main.py`**：新增對應的 `call_robot_api("POST", "/api/...")` 呼叫點；如果這個動作代表包裹狀態機的新階段，記得：
   - 在 `app/models.py` 的 `Package.status` 註解與程式邏輯中新增這個狀態；
   - 在 `TaskLog.event_type` 的註解清單補上新的事件類型字串；
   - 視需要在 Dashboard 的 `ADMIN_DASHBOARD_HTML`／`ADMIN_EXCEPTIONS_HTML` 加上對應的按鈕與 JS 呼叫。
6. 兩邊 README（`line-backend/README.md`、`flashbot-robot/README.md`）與 `flashbot-robot/REPORT.md` 的 API 清單建議同步更新，維持文件與程式碼一致（目前已經發現至少兩處文件落後於程式碼的案例，見 5.3 節）。

### 5.2 新增一個「Dashboard 監控數據」該改哪裡

- **如果是機器人硬體相關**（電量、位置、艙門狀態等）：
  1. 先確認 `FlashbotController.get_status_summary()`（`robot.py`）有沒有已經合併但沒有被用到的欄位；若沒有，擴充這支方法的回傳內容。
  2. `flashbot-robot/src/aurobox/api.py` 的 `GET /api/dashboard/status` 目前把 `live_status` 與 `door_states` 一起回傳，可以直接擴充這個 payload。
  3. `line-backend` 這邊的 `GET /admin/robot-status`（`app/main.py`）目前是「整包轉發」機器人回傳的 JSON，通常不用改；但 Dashboard 前端的 `loadRobotStatus()`（`ADMIN_DASHBOARD_HTML` 內的 JS）要跟著更新才會顯示出新欄位。
- **如果是包裹/業務相關數據**（例如新的統計指標）：
  1. 在 `line-backend/app/main.py` 新增或擴充 `/admin/packages/*` 系列 API（注意 `/admin/packages`（分頁）與 `/admin/packages/live`（即時、未分頁）用途不同，別把即時用的查詢改成全量掃描）。
  2. 如果這是值得留存歷史的事件，透過 `log_event()` 寫進 `TaskLog`，每日報表（`/admin/reports/daily`）會自動撈得到。
  3. 前端顯示邏輯在對應的 `ADMIN_*_HTML` 常數裡新增 DOM 元素 + JS 渲染函式。

### 5.3 測試與除錯注意事項

- **`flashbot-robot/tests/test_api_integration.py` 與 `tests/load_test.py` 目前疑似與現行 `api.py` 不同步 `[需補充：請確認並修正/更新測試]`**：
  這兩支測試呼叫的是 `POST /api/packages/{package_id}/assign`、`/api/packages/{id}/complete`、`/api/packages/{id}/cancel`，並假設 `Door` model 上有 `package_id` 欄位；但目前 `src/aurobox/api.py` 與 `src/aurobox/models.py` 的實際路由與欄位都已經改成以 `door_task_id` 為主（`/api/door-tasks/{door_task_id}/assign` 等），`Door` model 上也沒有 `package_id` 欄位。這代表目前這兩份測試很可能**在現行程式碼下無法通過或根本沒有測到真實路由**——建議優先處理，否則之後改動 `api.py` 時，這兩份測試起不了保護作用。
- **`flashbot-robot/REPORT.md` 記載的「pytest: 4 passed, 1 skipped」**與上一點的觀察有落差，建議實際重跑一次 `pytest`（`flashbot-robot` 目錄下）確認目前真實的測試結果，而不是只信任報告裡的舊數字。
- **`line-backend/README.md`「已知限制」章節提到的 `triggered_name` 未定義 bug，經比對目前 `app/main.py` 的 `handle_postback()` REJECT 分支，`triggered_name` 已經有正確賦值** —— 這代表該文件描述的問題**很可能已經修好但 README 忘記更新**，建議確認實際行為後同步修正 README，避免未來開發者被過時的「已知限制」誤導去重複修一個已經不存在的 bug。
- **`admin_robot_recall()` 的 docstring（`app/main.py`）寫「這裡只保護了叫回這一側，`place_package`／`admin_dispatch_batch` 目前還沒有加鎖」**，但實際程式碼中 `place_package()` 已使用 `db.refresh(package, with_for_update={"nowait": True})`、`admin_dispatch_batch()` 已使用 `.with_for_update(nowait=True)`。同樣是**程式碼已演進、註解未同步**的案例，建議一併更新該處註解，避免誤導後續開發者以為並發保護還沒做。
- **LINE 官方帳號的月推播額度**：若使用 LINE 官方帳號的免費方案，主動推播（push message）通常有每月則數上限，這是全系統高頻率使用 `push_message`（到貨通知、抵達通知、各種狀態更新）時容易撞到的營運瓶頸，非程式邏輯問題但會直接影響住戶體驗（收不到通知）。建議定期關注用量，評估升級方案或精簡非必要的 push（例如目前 `line_messaging.py` 中已被註解掉的部分推播即是精簡過的痕跡）。`[需補充：目前實際使用的 LINE 方案與月用量]`
- **兩個 LIFF App 是分開註冊、各自綁定一個固定的 Endpoint URL**（`LIFF_ID` → `/liff/scan`，`LIFF_ID_RETURN` → `/liff/return-request`），如果之後要換網域（例如從 ngrok 換成正式網域），記得回 LINE Developers 後台同步更新這兩個 LIFF App 的 Endpoint URL，否則掃碼/退貨頁面會連到舊網址。
- **本機開發除錯**：`line-backend` 用 `start-server.bat` + `start-ngrok.bat`（Windows，`cmd.exe`），啟動後可在 `http://localhost:8000/docs` 用 FastAPI 內建的 Swagger UI 互動測試所有 API；`flashbot-robot` 沒有對應的一鍵啟動 `.bat`，目前是 `python -u run.py --debug` 手動啟動，也可以用 `src/aurobox/cli.py` 提供的 CLI（`aurobox --sn <SN> status` 等）單獨測試 Pudu 連線，不需要整套 Flask app 起來。
- **併發保護的設計原則**：整個系統大量使用 `with_for_update(nowait=True)`（FastAPI 這邊，因為 handler 是同步函式跑在 async 事件迴圈中，用 `nowait` 避免卡住整個 event loop）與 `with_for_update(skip_locked=True)`（排程背景任務，鎖不到就跳過、下一輪再檢查）。**新增任何會修改 `Package`／`Door` 狀態的邏輯時，應延續同一套模式**，不要直接用不帶參數的 `with_for_update()` 傻等鎖，否則可能拖垮整個服務的回應時間。

---

*本文件由程式碼靜態分析產生，涵蓋 `line-backend`（`main` branch）與 `flashbot-robot`（`flashbot` branch）目前可見的原始碼、README、測試與設定檔。所有標註 `[需補充]` 之處，代表分析當下無法從程式碼 100% 確認，建議由你或團隊成員驗證後補上。*
