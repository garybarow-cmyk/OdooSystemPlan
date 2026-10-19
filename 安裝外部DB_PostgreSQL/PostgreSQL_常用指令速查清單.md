# PostgreSQL 常用指令速查清單 (Cheat Sheet)

本清單整理了在 Ubuntu 環境下管理 PostgreSQL 資料庫時**最常用、最核心的指令**，包含 Linux 系統服務、psql 終端指令、使用者權限、資料庫維護、備份還原與連線監控。

---

## 目錄
1. [Linux 系統服務管理](#1-linux-系統服務管理)
2. [資料庫登入與連線](#2-資料庫登入與連線)
3. [psql 終端環境專用指令 (反斜線語法)](#3-psql-終端環境專用指令-反斜線語法)
4. [使用者與權限管理 SQL](#4-使用者與權限管理-sql)
5. [資料庫與資料表管理 SQL](#5-資料庫與資料表管理-sql)
6. [資料庫大小與效能檢視](#6-資料庫大小與效能檢視)
7. [目前連線查詢與強制中斷](#7-目前連線查詢與強制中斷)
8. [資料庫備份與還原 (Backup & Restore)](#8-資料庫備份與還原-backup--restore)
9. [JSONB 資產規格常用查詢 (資產系統專用)](#9-jsonb-資產規格常用查詢-資產系統專用)

---

## 1. Linux 系統服務管理

在 Ubuntu 終端機執行的系統服務指令：

```bash
# 檢查 PostgreSQL 服務運作狀態
sudo systemctl status postgresql

# 啟動服務
sudo systemctl start postgresql

# 停止服務
sudo systemctl stop postgresql

# 重新啟動服務 (修改設定檔後常用)
sudo systemctl restart postgresql

# 重新載入設定檔 (不中斷現有連線)
sudo systemctl reload postgresql

# 設定開機自動啟動
sudo systemctl enable postgresql

# 查看最近的日誌紀錄 (排查啟動或連線錯誤)
sudo journalctl -u postgresql -n 50 --no-pager
```

---

## 2. 資料庫登入與連線

```bash
# 1. 以預設管理員 postgres 身分進入 psql
sudo -u postgres psql

# 2. 以特定帳號登入本機特定資料庫 (會提示輸入密碼)
psql -U <帳號> -d <資料庫名稱> -W

# 3. 遠端連線到外部 PostgreSQL 伺服器
psql -h <伺服器IP> -p 5432 -U <帳號> -d <資料庫名稱>

# 4. 不進入互動式終端，直接在 Linux 終端執行單一 SQL
psql -U <帳號> -d <資料庫名稱> -c "SELECT version();"
```

---

## 3. psql 終端環境專用指令 (反斜線語法)

進入 `psql` 後，最常使用的快捷指令：

| 快捷指令 | 說明 |
| :--- | :--- |
| `\q` | **退出** psql 終端環境 |
| `\l` 或 `\l+` | **列出所有資料庫**（加 `+` 顯示大小、擁有者等詳細資訊） |
| `\c <資料庫名稱>` | **切換**至指定資料庫（如 `\c asset_management_db`） |
| `\dt` 或 `\dt+` | **列出目前資料庫內的所有資料表** |
| `\d <資料表名稱>` | **檢視資料表結構**（欄位名稱、資料型態、預設值、索引、外鍵等） |
| `\du` 或 `\du+` | **列出所有使用者帳號（Roles）** 與其具備的權限 |
| `\di` | 列出目前資料庫內的所有**索引** |
| `\dn` | 列出所有的 Schema（命名空間） |
| `\x` | **切換展開模式 (Expanded display)**：欄位太多時轉為直式顯示（極實用！） |
| `\timing` | 開啟或關閉**查詢執行時間計時** |
| `\i <腳本路徑.sql>` | 在 psql 內直接**執行外部 SQL 腳本檔案** |
| `\?` | 查看所有 psql 反斜線指令說明 |
| `\h <SQL關鍵字>` | 查看特定 SQL 語法的語法說明（如 `\h SELECT`） |

---

## 4. 使用者與權限管理 SQL

在 `psql` 終端內執行：

```sql
-- 建立新使用者並設定密碼
CREATE USER <使用者名稱> WITH PASSWORD '<密碼>';

-- 修改現有使用者密碼
ALTER USER <使用者名稱> WITH PASSWORD '<新密碼>';

-- 賦予使用者超級管理員權限 (Superuser)
ALTER USER <使用者名稱> WITH SUPERUSER;

-- 授予資料庫的完整權限給使用者
GRANT ALL PRIVILEGES ON DATABASE <資料庫名稱> TO <使用者名稱>;

-- (PostgreSQL 15+ 重要) 授予使用者在 public schema 下建立資料表的權限
GRANT ALL ON SCHEMA public TO <使用者名稱>;

-- 授予特定資料庫中「所有現存資料表」的讀寫權限
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO <使用者名稱>;

-- 設定未來建立的資料表預設自動賦予權限
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO <使用者名稱>;

-- 刪除使用者 (需先移除其擁有的物件與權限)
DROP USER <使用者名稱>;
```

---

## 5. 資料庫與資料表管理 SQL

```sql
-- 建立新資料庫並指定擁有人
CREATE DATABASE <資料庫名稱> OWNER <擁有者帳號>;

-- 刪除資料庫 (注意：不可逆)
DROP DATABASE <資料庫名稱>;

-- 重新命名資料庫
ALTER DATABASE <舊名稱> RENAME TO <新名稱>;

-- 清空整張資料表的資料 (重設自增 ID)
TRUNCATE TABLE <資料表名稱> RESTART IDENTITY;

-- 刪除資料表
DROP TABLE IF EXISTS <資料表名稱> CASCADE;
```

---

## 6. 資料庫大小與效能檢視

```sql
-- 1. 查看各資料庫佔用硬碟空間大小 (由大到小排序)
SELECT 
    datname AS database_name, 
    pg_size_pretty(pg_database_size(datname)) AS size 
FROM pg_database 
ORDER BY pg_database_size(datname) DESC;

-- 2. 查看目前資料庫內各資料表的大小 (前 10 名)
SELECT 
    relname AS table_name, 
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size 
FROM pg_catalog.pg_statio_user_tables 
ORDER BY pg_total_relation_size(relid) DESC 
LIMIT 10;

-- 3. 手動觸發資料庫清理與統計分析 (釋放膨脹空間並更新查詢最佳化計畫)
VACUUM ANALYZE VERBOSE;
```

---

## 7. 目前連線查詢與強制中斷

排查資料庫被卡住（Lock）或連線過多時：

```sql
-- 1. 檢視目前有哪些連線與其執行的 SQL
SELECT 
    pid, 
    usename, 
    client_addr, 
    application_name, 
    state, 
    query, 
    query_start 
FROM pg_stat_activity 
WHERE state != 'idle';

-- 2. 取消某個連線正在執行的查詢 (溫和中斷)
SELECT pg_cancel_backend(<PID>);

-- 3. 強制中斷並中斷該連線 (強制踢出)
SELECT pg_terminate_backend(<PID>);

-- 4. 踢出特定資料庫的所有外部連線 (準備刪庫維護時必用)
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE datname = '<資料庫名稱>' AND pid <> pg_backend_pid();
```

---

## 8. 資料庫備份與還原 (Backup & Restore)

此區指令在 **Linux 終端機**（非 psql）執行：

### 備份 (Dump)
```bash
# 1. 備份單一資料庫為純 SQL 文字檔
pg_dump -U <帳號> -h localhost -d <資料庫名稱> > backup_$(date +%Y%m%d).sql

# 2. 備份單一資料庫為 Custom 壓縮格式 (推薦，支援多執行緒還原與篩選)
pg_dump -U <帳號> -h localhost -F c -b -v -f backup_$(date +%Y%m%d).dump <資料庫名稱>

# 3. 備份整個 PostgreSQL 伺服器 (包含所有資料庫與使用者帳號權限)
pg_dumpall -U postgres > full_backup_$(date +%Y%m%d).sql
```

### 還原 (Restore)
```bash
# 1. 從純文字 SQL 備份還原
psql -U <帳號> -d <資料庫名稱> < backup_20261016.sql

# 2. 從 Custom 壓縮格式還原 (使用 pg_restore)
pg_restore -U <帳號> -d <資料庫名稱> -v backup_20261016.dump

# 3. 還原時先清除現有資料表結構 (--clean)
pg_restore -U <帳號> -d <資料庫名稱> --clean -v backup_20261016.dump
```

---

## 9. JSONB 資產規格常用查詢 (資產系統專用)

針對資產管理系統中存放在 `specifications` 欄位內的動態規格進行查詢：

```sql
-- 1. 取出特定屬性文字 (使用 ->> 運算子)
SELECT asset_code, name, specifications->>'cpu' AS cpu_spec 
FROM assets;

-- 2. 依據 JSON 內部屬性進行條件篩選
SELECT asset_code, name 
FROM assets 
WHERE specifications->>'cpu' ILIKE '%Intel%';

-- 3. 數值型態轉型與比對 (篩選 RAM >= 32GB)
SELECT asset_code, name, (specifications->>'ram_gb')::int AS ram
FROM assets 
WHERE (specifications->>'ram_gb')::int >= 32;

-- 4. 包含特定鍵值組合 (使用 @> 運算子，能走 GIN 索引，效能極高)
SELECT * FROM assets 
WHERE specifications @> '{"status": "active", "ram_gb": 64}';

-- 5. 檢查某個鍵 (Key) 是否存在 (使用 ? 運算子)
SELECT asset_code FROM assets 
WHERE specifications ? 'mac_address';
```
