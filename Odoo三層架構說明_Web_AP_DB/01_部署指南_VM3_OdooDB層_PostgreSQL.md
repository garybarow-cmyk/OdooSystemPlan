# 01. VM 3: Odoo DB 層（PostgreSQL 16）部署指南

本指南說明如何在獨立的 **VM 3 (Ubuntu 26)** 上原生安裝與調校 **PostgreSQL 16**，專供 Odoo 18 AP 伺服器進行高併發連線與交易處理。

---

## 一、 主機資訊與網路規劃範例

| 項目 | 範例設定值 | 說明 |
| :--- | :--- | :--- |
| **主機角色** | Odoo 核心資料庫 (Odoo DB) | 專職處理 ERP 核心資料庫存取 |
| **作業系統** | Ubuntu 26.04 LTS | 建議最小化安裝（無桌面環境） |
| **建議主機 IP** | `192.168.1.130` | 僅供內部區網存取，不可對外公開 |
| **預設 Port** | `5432` | 僅允許 AP 伺服器連線 |
| **Odoo 專用帳號** | `odoo` | 擁有建立資料庫權限的專用帳號 |
| **建議硬體** | 4 vCPU / 8GB ~ 16GB RAM / 100GB+ SSD | 磁碟 I/O 越高越好 |

---

## 二、 部署步驟

### 步驟 1：系統更新與安裝 PGDG 官方來源庫

```bash
# 1. 更新系統
sudo apt update && sudo apt upgrade -y

# 2. 安裝金鑰管理工具
sudo apt install -y curl ca-certificates gnupg lsb-release

# 3. 加入 PostgreSQL 官方金鑰與 APT 來源
sudo install -d /etc/apt/keyrings
curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo gpg --dearmor -o /etc/apt/keyrings/postgresql.gpg
echo "deb [signed-by=/etc/apt/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list

# 4. 更新套件清單並安裝 PostgreSQL 16
sudo apt update
sudo apt install -y postgresql-16 postgresql-contrib-16

# 5. 開機自動啟動
sudo systemctl enable postgresql
sudo systemctl status postgresql
```

---

### 步驟 2：建立 Odoo 專用資料庫使用者

Odoo 在初次啟動時需要有權限自動建立資料庫（`createdb`）：

```bash
# 切換為 postgres 管理員建立 odoo 使用者
sudo -u postgres createuser -s -P odoo
```
> 系統會提示您輸入密碼（例如設定為 `OdooDbStrongPass123`），請妥善記住，稍後 AP 層設定會用到。

---

### 步驟 3：設定外部監聽與網路安全白名單

#### 1. 修改 `postgresql.conf`（開放網路監聽）
```bash
sudo nano /etc/postgresql/16/main/postgresql.conf
```
找到 `listen_addresses`，修改為監聽所有網卡：
```conf
listen_addresses = '*'
port = 5432
```

#### 2. Odoo 專屬效能調校（同在 `postgresql.conf` 內修改，以 8GB RAM 為例）
```conf
# 記憶體調校
shared_buffers = 2GB                  # 約佔總 RAM 的 25%
effective_cache_size = 6GB            # 約佔總 RAM 的 50% ~ 75%
work_mem = 64MB                       # 複雜排序/Hash 運算記憶體
maintenance_work_mem = 512MB          # 索引重建與 Vacuum 記憶體
max_connections = 120                 # 支援多個 AP Worker 連線

# WAL 與磁碟寫入調校
wal_buffers = 16MB
checkpoint_completion_target = 0.9
random_page_cost = 1.1                # SSD 固態硬碟最佳化參數
```
*儲存並退出（`Ctrl + O` -> `Enter` -> `Ctrl + X`）。*

---

#### 3. 修改 `pg_hba.conf`（嚴格白名單：僅放行 AP VM）
```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```
在檔案最底部加入一條規則（**假設 AP 伺服器 IP 為 `192.168.1.120`**）：
```conf
# TYPE  DATABASE        USER  ADDRESS            METHOD
host    all             odoo  192.168.1.120/32   scram-sha-256
```
> ⚠️ **安全重點**：請勿設定 `0.0.0.0/0`，指定 AP VM 的精準 IP (`/32`) 能防止內網其他機器任意嘗試連線。

---

### 步驟 4：設定 Ubuntu UFW 防火牆

```bash
# 允許來自 AP 伺服器 IP 存取 5432 埠
sudo ufw allow from 192.168.1.120 to any port 5432 proto tcp

# 允許管理員 SSH
sudo ufw allow 22/tcp

# 啟用並重新載入防火牆
sudo ufw enable
sudo ufw reload
sudo ufw status
```

---

### 步驟 5：重啟服務與驗證

```bash
# 重啟 PostgreSQL 套用參數
sudo systemctl restart postgresql

# 驗證監聽狀態是否為 0.0.0.0:5432
ss -nlt | grep 5432
```
若輸出 `LISTEN 0 244 0.0.0.0:5432`，即代表資料庫層部署完成，已就緒等待 AP 連線！
