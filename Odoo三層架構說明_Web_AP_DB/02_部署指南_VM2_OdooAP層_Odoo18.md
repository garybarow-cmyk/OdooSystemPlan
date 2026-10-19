# 02. VM 2: Odoo AP 層（Odoo 18）Docker 部署指南

本指南說明如何在獨立的 **VM 2 (Ubuntu 26)** 上使用 **Docker** 容器化部署 **Odoo 18** 應用伺服器，連接 VM 3 的 PostgreSQL 資料庫，並開放自訂模組（如資產管理串接模組）的熱插拔掛載。

---

## 一、 主機資訊與網路規劃範例

| 項目 | 範例設定值 | 說明 |
| :--- | :--- | :--- |
| **主機角色** | Odoo 18 應用伺服器 (Odoo AP) | 專職處理商業邏輯、報表產製與客製模組 |
| **作業系統** | Ubuntu 26.04 LTS | 需安裝 Docker Engine 與 Compose |
| **建議主機 IP** | `192.168.1.120` | 僅供內部區網存取，不可對外公開 |
| **監聽 Ports** | `8069` (HTTP) / `8072` (長輪詢/Websocket) | 僅允許 VM 1 (Web) 連線 |
| **連線後端 DB** | `192.168.1.130:5432` | 連線至 VM 3 (Odoo DB) |
| **建議硬體** | 4 vCPU / 8GB RAM / 60GB SSD | 核心數決定並行 Worker 數量 |

---

## 二、 部署步驟

### 步驟 1：安裝 Docker 與 Docker Compose 套件

在 VM 2 終端機執行：

```bash
# 1. 更新並安裝前置工具
sudo apt update && sudo apt install -y ca-certificates curl gnupg

# 2. 加入 Docker 官方 GPG 金鑰與 Repository
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. 安裝 Docker 引擎與 Compose 插件
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 4. 允許目前使用者免 sudo 執行 docker（可選）
sudo usermod -aG docker $USER
```

---

### 步驟 2：規劃 Odoo 目錄結構

建立專屬工作目錄，將設定檔、自訂模組、附檔分開存放：

```bash
sudo mkdir -p /opt/odoo18/{config,addons,filestore}
cd /opt/odoo18
```

目錄功能說明：
* `config/`：存放 `odoo.conf` 設定檔。
* `addons/`：存放您自己開發或第三方的自訂模組（例如「資產管理串接模組」）。
* `filestore/`：存放 Odoo 所有的附件、圖片與 PDF 文件檔案。

---

### 步驟 3：建立 Odoo 設定檔 (`config/odoo.conf`)

建立設定檔：
```bash
sudo nano /opt/odoo18/config/odoo.conf
```

寫入以下內容（請注意修改資料庫 IP 與密碼）：

```ini
[options]
; 資料庫連線設定 (指向 VM 3: Odoo DB)
db_host = 192.168.1.130
db_port = 5432
db_user = odoo
db_password = OdooDbStrongPass123
db_maxconn = 64

; 模組路徑 (包含系統自帶與自訂目錄)
addons_path = /usr/lib/python3/dist-packages/odoo/addons,/mnt/extra-addons

; 附件儲存路徑
data_dir = /var/lib/odoo

; 反向代理關鍵參數 (因為前面有 Nginx，必須設為 True)
proxy_mode = True

; 多行程 (Workers) 效能調校 (建議為 CPU核心數 * 2 + 1)
workers = 5

; 記憶體與逾時時間保護 (避免慢查詢拖垮服務)
limit_memory_hard = 2684354560
limit_memory_soft = 2147483648
limit_time_cpu = 600
limit_time_real = 1200
limit_request = 8192

; 系統日誌
log_level = info
```
*儲存並退出（`Ctrl + O` -> `Enter` -> `Ctrl + X`）。*

---

### 步驟 4：建立 `docker-compose.yml`

在 `/opt/odoo18/` 目錄下建立編排檔：
```bash
sudo nano /opt/odoo18/docker-compose.yml
```

寫入以下配置：

```yaml
services:
  odoo:
    image: odoo:18.0
    container_name: odoo18_app
    restart: always
    ports:
      - "8069:8069"   # HTTP 業務流量
      - "8072:8072"   # Websocket / 即時通訊流量
    environment:
      - HOST=192.168.1.130
      - USER=odoo
      - PASSWORD=OdooDbStrongPass123
    volumes:
      - ./config/odoo.conf:/etc/odoo/odoo.conf:ro
      - ./addons:/mnt/extra-addons
      - ./filestore:/var/lib/odoo
```

---

### 步驟 5：配置目錄權限

官方 Odoo Docker 映像檔內部運行的使用者 UID 為 `101`，必須賦予目錄寫入權限：

```bash
sudo chown -R 101:101 /opt/odoo18/addons /opt/odoo18/filestore
```

---

### 步驟 6：啟動 Odoo 18 容器

```bash
cd /opt/odoo18

# 背景啟動
sudo docker compose up -d

# 即時檢視啟動日誌 (按 Ctrl + C 離開)
sudo docker compose logs -f
```
> 若日誌最後出現 `HTTP service (werkzeug) running on 0.0.0.0:8069`，代表 Odoo 已成功啟動並連上 VM 3 的資料庫！

---

### 步驟 7：設定 Ubuntu UFW 防火牆

僅允許 **VM 1 (Web Nginx)** 存取 8069 與 8072 埠：

```bash
# 允許來自 Web VM IP (例如 192.168.1.110) 存取
sudo ufw allow from 192.168.1.110 to any port 8069 proto tcp
sudo ufw allow from 192.168.1.110 to any port 8072 proto tcp

# 允許 SSH
sudo ufw allow 22/tcp

# 啟用防火牆
sudo ufw enable
sudo ufw reload
sudo ufw status
```
