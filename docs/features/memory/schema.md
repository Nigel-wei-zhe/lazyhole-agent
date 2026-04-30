# Memory SQLite Schema

路徑：`~/.lazyhole/memory.sqlite`，可用 `MEMORY_DB_PATH` 覆蓋。初始化由 `src/utils/memory-db.js` 的 `ensureDb()` 執行。

## 資料流

```txt
session 結束
  -> memory_archives
  -> memory_reviews(status=pending)
  -> keep/edit 後寫入 memory_facts
```

## memory_archives

原始封存摘要。append-only，用於回溯與搜尋，不直接代表會注入 Agent 的長期記憶。

| 欄位 | 型別 | 說明 |
|---|---|---|
| `id` | INTEGER PK | 歸檔流水號 |
| `session_id` | TEXT | 來源 session UUID |
| `chat_id` | TEXT | Telegram chat |
| `user_id` | TEXT | Telegram user，可空 |
| `started_at` | TEXT | session 建立時間 |
| `ended_at` | TEXT | session 結束時間 |
| `archived_at` | TEXT | 歸檔時間 |
| `trigger` | TEXT | `clear` / `end_session` / `ttl` |
| `active_skill` | TEXT | 歸檔時的 active skill |
| `category` | TEXT | 預留分類 |
| `summary` | TEXT | LLM 產生的長期摘要 |
| `raw_chars` | INTEGER | 原始 session 字元數 |
| `history_count` | INTEGER | 原始 history 筆數 |
| `metadata_json` | TEXT | 擴充 metadata JSON |

索引：

- `idx_memory_archives_chat_archived(chat_id, archived_at DESC)`
- `idx_memory_archives_session(session_id)`

## memory_reviews

整理狀態。每次新增 archive 時同步建立 `pending` review；既有 archive 會在 `ensureDb()` 時補建 pending review。

| 欄位 | 型別 | 說明 |
|---|---|---|
| `archive_id` | INTEGER PK/FK | 對應 `memory_archives.id` |
| `chat_id` | TEXT | Telegram chat，便於 inbox 查詢 |
| `status` | TEXT | `pending` / `kept` / `dropped` / `edited` |
| `note` | TEXT | 人工整理備註 |
| `tags` | TEXT | 標籤字串，格式暫不約束 |
| `reviewed_at` | TEXT | 完成整理時間；pending 時可空 |
| `created_at` | TEXT | review 建立時間 |
| `updated_at` | TEXT | review 更新時間 |

索引：

- `idx_memory_reviews_chat_status(chat_id, status, updated_at DESC)`

## memory_facts

已確認長期記憶。未來 prompt 注入應讀此表，而不是直接讀 `memory_archives`。

| 欄位 | 型別 | 說明 |
|---|---|---|
| `id` | INTEGER PK | fact 流水號 |
| `chat_id` | TEXT | Telegram chat |
| `source_archive_id` | INTEGER FK | 來源 archive；手動新增可空 |
| `content` | TEXT | 會注入 Agent 的記憶內容 |
| `tags` | TEXT | 標籤字串，格式暫不約束 |
| `created_at` | TEXT | 建立時間 |
| `updated_at` | TEXT | 更新時間 |
| `deleted_at` | TEXT | 軟刪除時間；非空代表不注入 |

索引：

- `idx_memory_facts_chat_active(chat_id, deleted_at, updated_at DESC)`
- `idx_memory_facts_source_archive(source_archive_id)`

## 使用原則

| 表 | 用途 | 是否注入 Agent |
|---|---|---|
| `memory_archives` | 保存發生過什麼 | 否 |
| `memory_reviews` | 管理整理狀態 | 否 |
| `memory_facts` | 保存以後要記得什麼 | 是 |
