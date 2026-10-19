# Ubuntu 環境安裝外部 PostgreSQL 資料庫完整指南（供 Odoo 18 資產管理模組串接）

本文件提供在獨立的 **Ubuntu VM** 上安裝、配置 **PostgreSQL** 資料庫的完整步驟，專供 **Odoo 18** 模組透過網路跨機連線，並預設以**資產管理系統**為資料庫規劃核心。

---

## 目錄
1. [環境規劃建議](#1-環境規劃建議)
2. [步驟一：更新系統與前置相依套件](#步驟一更新系統與前置相依套件)
3. [步驟二：設定 PostgreSQL 官方 PGDG APT 來源庫](#步驟二設定-postgresql-官方-pgdg-apt-來源庫)
4. [步驟三：安裝 PostgreSQL 伺服器](#步驟三安裝-postgresql-伺服器)
5. [步驟四：建立資產管理專用資料庫與帳號](#步驟四建立資產管理專用資料庫與帳號)
6. [步驟五：設定外部網路連線（允許 Odoo 存取）](#步驟五設定外部網路連線允許-odoo-存取)
7. [步驟六：設定 Ubuntu 防火牆 (UFW)](#步驟六設定-ubuntu-防火牆-ufw)
8. [步驟七：連線測試與驗證](#步驟七連線測試與驗證)
9. [附錄 A：資產管理系統資料表初使化範例 (含 JSONB)](#附錄-a資產管理系統資料表初使化範例-含-jsonb)
10. [附錄 B：Odoo 18 模組連線程式碼範例](#附錄-bodoo-18-模組連線程式碼範例)

---

## 1. 環境規劃建議

| 項目 | 建議規格 / 設定 |
| :--- | :--- |
| **作業系統** | Ubuntu Server 24.04 LTS / 26.04 LTS |
| **資料庫版本** | **PostgreSQL 16**（兼具極高穩定性與 JSONB 效能） |
| **預設服務埠號 (Port)** | `5432` |
| **資料庫名稱** | `asset_management_db` |
| **專用帳號** | `asset_user` |
| **VM 建議硬體** | 2 vCPU / 4GB RAM / 40GB SSD 以上 |

---

## 步驟一：更新系統與前置相依套件

登入 Ubuntu VM 終端機，執行以下命令：

```bash
# 1. 更新現有套件清單與升級
sudo apt update && sudo apt upgrade -y

# 2. 安裝常見工具與金鑰處理套件
sudo apt install -y curl ca-certificates gnupg lsb-release
```

---

## 步驟二：設定 PostgreSQL 官方 PGDG APT 來源庫

使用官方儲存庫可確保無論 Ubuntu 版本為何，都能取得最新、修補最完整的 PostgreSQL 版本：

```bash
# 1. 建立金鑰存放目錄
sudo install -d /etc/apt/keyrings

# 2. 下載 PostgreSQL 官方 GPG 簽署金鑰
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/keyrings/postgresql.gpg

# 3. 加入官方 APT 來源庫清單
echo "deb [signed-by=/etc/apt/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list

# 4. 再次更新套件清單
sudo apt update
```

---

## 步驟三：安裝 PostgreSQL 伺服器

建議安裝 **PostgreSQL 16** 及延伸工具包：

```bash
# 安裝 PostgreSQL 16 核心與外掛延伸套件
sudo apt install -y postgresql-16 postgresql-contrib-16

# 檢查服務狀態
sudo systemctl status postgresql

# 設定開機自動啟動
sudo systemctl enable postgresql
```

---

## 步驟四：建立資產管理專用資料庫與帳號

切換為 `postgres` 系統管理員身分進入資料庫命令列：

```bash
sudo -u postgres psql
```

在 `postgres=#` 提示字元下，執行以下 SQL 指令：

```sql
-- 1. 建立資產管理系統專用帳號（請將 YOUR_STRONG_PASSWORD 替換為實際安全密碼）
CREATE USER asset_user WITH PASSWORD 'YOUR_STRONG_PASSWORD';
# asset_user請改成資產系統專用帳號名稱  //名稱自訂
# YOUR_STRONG_PASSWORD請改成資產系統專用帳號密碼//密碼自訂


-- 2. 建立資產管理專屬資料庫，並將擁有者設為 asset_user
CREATE DATABASE asset_management_db OWNER asset_user;
# asset_management_db請改成資產系統專用資料庫名稱 //名稱自訂
# asset_user請改成資產系統專用帳號名稱 //名稱自訂

-- 3. 授予權限
GRANT ALL PRIVILEGES ON DATABASE asset_management_db TO asset_user;
# asset_management_db請改成資產系統專用資料庫名稱 //名稱自訂
# asset_user請改成資產系統專用帳號名稱 //名稱自訂

-- 4. 退出 psql
\q
```

---

## 步驟五：設定外部網路連線（允許 Odoo 存取）

PostgreSQL 預設只監聽 `localhost`（本機），必須修改設定檔才能允許外部 Odoo 主機透過網路連入。

### 1. 修改監聽位址 (`postgresql.conf`)

編輯設定檔：
```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```

找到 `listen_addresses` 區段（大約在第 60 行附近），取消註解並修改為 `'*'`：
```conf
listen_addresses = '*'          # 監聽所有網卡介面
port = 5432                   # 埠號保持預設
```
*儲存並退出（Nano 編輯器：`Ctrl + O` 存檔，`Enter`，`Ctrl + X` 退出）。*

---

### 2. 開放存取白名單 (`pg_hba.conf`)

編輯客戶端驗證設定檔：
```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

移動到檔案最底部，加入允許 Odoo 主機連線的規則：

```conf
# TYPE  DATABASE             USER        ADDRESS            METHOD

# 方案 A (推薦 - 安全)：僅允許 Odoo VM 的特定 IP 連入（假設 Odoo IP 為 192.168.1.100）
host    asset_management_db  asset_user  192.168.1.100/32   scram-sha-256

# 方案 B (便利 - 內網網段)：允許內部整個區網網段連入（例如 192.168.1.0/24）
# host    asset_management_db  asset_user  192.168.1.0/24     scram-sha-256

# 方案 C (測試用 - 開放所有來源 IP，正式環境請勿使用)：
# host    asset_management_db  asset_user  0.0.0.0/0          scram-sha-256
```

---

## 步驟六：設定 Ubuntu 防火牆 (UFW)

如果 Ubuntu 虛擬機啟用了 `ufw` 防火牆，需放行 `5432` 埠：

```bash
# 僅允許 Odoo VM IP 存取 5432 埠（推薦）
sudo ufw allow from 192.168.1.100 to any port 5432 proto tcp

# 或者開放給整段內網：
# sudo ufw allow from 192.168.1.0/24 to any port 5432 proto tcp

# 重新載入防火牆規則
sudo ufw reload

# 查看狀態
sudo ufw status
```

---

## 步驟七：連線測試與驗證

### 1. 重啟 PostgreSQL 服務以套用設定
```bash
sudo systemctl restart postgresql
```

### 2. 本機連線驗證
```bash
psql -h localhost -U asset_user -d asset_management_db
```
輸入剛才設定的密碼，若成功進入 `asset_management_db=>` 即代表帳號設定無誤。輸入 `\q` 退出。

### 3. 從 Odoo VM 測試網路連線
在您的 **Odoo 主機** 上執行以下指令進行檢測：
```bash
# 測試 Port 是否通暢（需安裝 netcat）
nc -zv <Ubuntu_DB_VM_IP> 5432

# 或是直接使用 psql 客戶端連線測試
psql -h <Ubuntu_DB_VM_IP> -U asset_user -d asset_management_db
```

---

## 附錄 A：資產管理系統資料表初使化範例 (含 JSONB)

您可以先在 `asset_management_db` 中建立基礎表格，善用 PostgreSQL 的 **JSONB** 存放各種設備的不同規格：

登入資料庫：
```bash
sudo -u postgres psql -d asset_management_db
```

執行 SQL 建立 Schema：
```sql
-- 1. 建立資產主表
CREATE TABLE assets (
    id SERIAL PRIMARY KEY,
    asset_code VARCHAR(50) UNIQUE NOT NULL,      -- 資產編號/條碼 (如: AST-2026-0001)
    name VARCHAR(150) NOT NULL,                  -- 資產名稱
    category VARCHAR(50) NOT NULL,               -- 類別 (電腦設備/公務車/辦公耗材)
    status VARCHAR(20) DEFAULT 'available',      -- 狀態 (in_use, available, maintenance, scrapped)
    cost NUMERIC(12, 2) DEFAULT 0.00,           -- 取得成本
    purchase_date DATE,                          -- 採購日期
    custodian_id VARCHAR(50),                    -- 目前保管人/員工代號
    department VARCHAR(50),                      -- 所屬部門
    specifications JSONB DEFAULT '{}'::jsonb,    -- 動態詳細規格 (CPU, RAM, MAC, 車牌等)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. 為 JSONB 欄位建立 GIN 索引（加速快速搜尋內部屬性）
CREATE INDEX idx_assets_specifications ON assets USING gin (specifications);

-- 3. 建立資產異動/借還日誌表
CREATE TABLE asset_logs (
    id SERIAL PRIMARY KEY,
    asset_id INT REFERENCES assets(id) ON DELETE CASCADE,
    action_type VARCHAR(30) NOT NULL,            -- 動作 (領用、歸還、維修、盤點)
    operator VARCHAR(50) NOT NULL,               -- 操作人員
    details TEXT,                                -- 備註與說明
    log_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 4. 插入一筆測試資產資料
INSERT INTO assets (asset_code, name, category, status, cost, purchase_date, specifications)
VALUES (
    'AST-SRV-001',
    '專案應用伺服器 1號機',
    'IT設備',
    'in_use',
    85000.00,
    '2026-01-15',
    '{"cpu": "Intel Xeon 16-Core", "ram_gb": 64, "storage": "2TB NVMe", "ip": "192.168.1.50"}'::jsonb
);
```

---

## 附錄 B：Odoo 18 模組連線程式碼範例

在 Odoo 18 的自訂模組 Python 檔案中（例如 `models/asset_connector.py`），可以直接透過 `psycopg2` 連線到外部 Ubuntu DB：

```python
import psycopg2
from psycopg2.extras import RealDictCursor
from odoo import models, api, fields
import logging

_logger = logging.getLogger(__name__)

class ExternalAssetService(models.AbstractModel):
    _name = 'external.asset.service'
    _description = '外部資產資料庫串接服務'

    def _get_db_connection(self):
        """建立與外部 Ubuntu PostgreSQL 的連線"""
        return psycopg2.connect(
            host="192.168.1.200",             # 外部 Ubuntu VM 的 IP
            port=5432,
            dbname="asset_management_db",
            user="asset_user",
            password="YOUR_STRONG_PASSWORD",
            connect_timeout=5
        )

    def fetch_assets(self, limit=50):
        """從外部資產系統獲取資產清單"""
        try:
            with self._get_db_connection() as conn:
                with conn.cursor(cursor_factory=RealDictCursor) as cur:
                    query = """
                        SELECT id, asset_code, name, category, status, cost, specifications
                        FROM assets
                        ORDER BY id DESC
                        LIMIT %s;
                    """
                    cur.execute(query, (limit,))
                    results = cur.fetchall()
                    return results
        except Exception as e:
            _logger.error("連線外部資產資料庫失敗: %s", str(e))
            return []
```
