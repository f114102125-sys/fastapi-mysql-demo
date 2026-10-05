# Todo 任務清單

FastAPI + SQLAlchemy + MySQL 的 Todo 專案，附繁體中文網頁介面。支援註冊、帳密登入、登出，以及每位使用者獨立的任務清單。

## 安裝與啟動

需要 Python 3.10+ 與 MySQL。以下以 PowerShell 為例：

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

先在 MySQL 執行 `fastapi_db.sql` 建立資料庫；也可以只執行 `CREATE DATABASE fastapi_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`。啟動時會建立 `users`、`todos` 表；舊版 `todos` 若沒有 `owner_id`，啟動時會自動新增該欄位，保留原資料。其他欄位變動仍需另外遷移。

設定資料庫連線（參考 `.env.example`）：

```powershell
$env:DATABASE_URL = "mysql+pymysql://root:your_password@localhost:3306/fastapi_db"
$env:SESSION_SECRET_KEY = "請換成一段夠長且隨機的字串"
python -m uvicorn app.main:app --reload
```

連線設定透過環境變數讀取；若未設定，預設為 `mysql+pymysql://root:@localhost:3306/fastapi_db`，供本機無密碼的 MySQL 練習環境使用。`SESSION_SECRET_KEY` 用於簽署登入 Cookie；未設定時每次啟動會產生新金鑰，重啟後須重新登入。正式部署請設定固定的隨機金鑰、使用 HTTPS，並設定 `SESSION_COOKIE_SECURE=true`，讓 Cookie 只透過 HTTPS 傳送。此專案不會自動載入 `.env` 檔案。

開啟首頁後可直接註冊帳號（3–50 字元英數或底線，密碼至少 8 字元），密碼以 PBKDF2 雜湊存入 MySQL 的 `users` 表。舊任務的 `owner_id` 為 NULL，不會顯示於任何帳號；註冊後可用管理工具查詢 `SELECT id, username FROM users;`，再執行 `UPDATE todos SET owner_id = <使用者id> WHERE owner_id IS NULL;` 指派舊任務。若有多位使用者，請先確認各任務的歸屬再更新。

- 網頁：<http://127.0.0.1:8000/>
- 互動式 API 文件：<http://127.0.0.1:8000/docs>
- 舊的 `python main.py` 或 `uvicorn main:app --reload` 啟動方式仍可使用。

## API

| 方法 | 路徑 | 說明 |
| --- | --- | --- |
| POST | `/auth/register` | 註冊，JSON `{"username":"alice","password":"password123"}` |
| POST | `/auth/login` | 登入，格式同上 |
| POST | `/auth/logout` | 登出 |
| GET | `/todos/?skip=0&limit=100` | 依最新建立順序列出任務 |
| GET | `/todos/{todo_id}` | 讀取任務 |
| POST | `/todos/` | 新增任務，例如 `{"title":"買牛奶"}` |
| PATCH | `/todos/{todo_id}` | 修改任務，例如 `{"completed":true}` 或 `{"title":"新名稱"}` |
| DELETE | `/todos/{todo_id}` | 刪除任務 |

任務 API 須先登入；修改資料的 POST/PATCH/DELETE 請求需附上頁面中的 CSRF token（`X-CSRF-Token` header）。網頁會自動處理登入 Cookie 與驗證碼。

## 專案結構

- `app/core/`：環境設定
- `app/db/`、`app/models/`：連線與資料表模型
- `app/schemas/`：API 輸入輸出驗證
- `app/repositories/`：資料庫操作
- `app/api/`：Todo API 路由
- `app/web/`、`app/templates/`、`app/static/`：頁面及靜態檔案
- `tests/`：API 測試

執行測試：`python -m pip install -r requirements-dev.txt` 後執行 `python -m pytest`。測試使用記憶體 SQLite，不會操作 MySQL 資料。
