# 03. VM 1: Odoo Web 層（Nginx 反向代理）Docker 部署指南

本指南說明如何在獨立的 **VM 1 (Ubuntu 26)** 上使用 **Docker** 容器化部署 **Nginx** 反向代理伺服器，負責對外暴露安全的 HTTPS (443) 服務、終端解密 SSL、快取靜態資源，並精準分流業務請求與 Websocket 即時長輪詢至 VM 2 (Odoo AP)。

---

## 一、 主機資訊與網路規劃範例

| 項目 | 範例設定值 | 說明 |
| :--- | :--- | :--- |
| **主機角色** | Web 反向代理與閘道 (Odoo Web) | 唯一對外直接公開的伺服器 |
| **作業系統** | Ubuntu 26.04 LTS | 需安裝 Docker Engine 與 Compose |
| **對外公開 IP** | 外部實體 IP / 網域名稱 | 例如 `erp.yourcompany.com` |
| **對內網卡 IP** | `192.168.1.110` | 用於向內網 AP 溝通 |
| **對外開放 Ports** | `80` (HTTP) 與 `443` (HTTPS) | 供使用者瀏覽器連線存取 |
| **轉發後端 AP** | `192.168.1.120:8069` 及 `8072` | 轉發至 VM 2 (Odoo AP) |
| **建議硬體** | 2 vCPU / 2GB ~ 4GB RAM / 30GB SSD | 記憶體需求較小，重視網路頻寬 |

---

## 二、 部署步驟

### 步驟 1：安裝 Docker 與 Docker Compose

在 VM 1 終端機執行：

```bash
# 1. 更新並安裝相依工具
sudo apt update && sudo apt install -y ca-certificates curl gnupg

# 2. 加入 Docker 官方 GPG 金鑰與 APT 清單
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 3. 安裝 Docker 引擎
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

---

### 步驟 2：規劃 Nginx 目錄結構

```bash
sudo mkdir -p /opt/nginx-odoo/{conf.d,certs}
cd /opt/nginx-odoo
```

* `conf.d/`：存放 Nginx 的 Odoo 轉發規則設定檔。
* `certs/`：存放 SSL 憑證（`fullchain.pem` 與 `privkey.pem`）。

---

### 步驟 3：產生 SSL 憑證

* **情況 A（測試或內網環境）**：快速自簽一組 10 年期 SSL 測試憑證：
  ```bash
  sudo openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
    -keyout /opt/nginx-odoo/certs/privkey.pem \
    -out /opt/nginx-odoo/certs/fullchain.pem \
    -subj "/C=TW/ST=Taiwan/L=Taipei/O=Company/CN=erp.local"
  ```

* **情況 B（正式公網網域名稱）**：將您購買或 Let's Encrypt 的憑證檔放入 `/opt/nginx-odoo/certs/`，檔名對應為 `fullchain.pem` 與 `privkey.pem`。

---

### 步驟 4：撰寫 Nginx 轉發設定檔 (`conf.d/odoo.conf`)

建立設定檔：
```bash
sudo nano /opt/nginx-odoo/conf.d/odoo.conf
```

寫入以下完整設定（**請注意將後端 IP 改為 VM 2 的 IP `192.168.1.120`**）：

```nginx
# 1. 定義上游 Odoo AP 服務 (指向 VM 2)
upstream odoo_backend {
    server 192.168.1.120:8069;
}

upstream odoo_chat {
    server 192.168.1.120:8072;
}

# 2. HTTP 80 自動轉址為 HTTPS 443
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

# 3. HTTPS 443 主服務設定
server {
    listen 443 ssl;
    server_name _;

    # SSL 憑證路徑 (掛載於容器內部)
    ssl_certificate /etc/nginx/certs/fullchain.pem;
    ssl_certificate_key /etc/nginx/certs/privkey.pem;

    # SSL 加密安全通訊協定
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    # 效能與逾時保護設定
    proxy_read_timeout 720s;
    proxy_connect_timeout 720s;
    proxy_send_timeout 720s;
    client_max_body_size 100M;     # 允許上傳大檔案 (附件/報表)

    # 啟用 Gzip 壓縮 (加速前端網頁載入)
    gzip on;
    gzip_types text/css text/less text/plain text/xml application/xml application/json application/javascript;

    # 轉發標頭 (轉遞客戶端真實 IP 給 Odoo)
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;

    # 規則 A: 即時通訊與推播 (Websocket / Longpolling)
    location /websocket {
        proxy_pass http://odoo_chat;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_redirect off;
    }

    # 規則 B: 一般業務操作與頁面轉發
    location / {
        proxy_pass http://odoo_backend;
        proxy_redirect off;
    }

    # 規則 C: 靜態資源本端快取 (防護 Odoo AP)
    location ~* /web/static/ {
        proxy_cache_valid 200 90m;
        proxy_buffering on;
        expires 864000;
        proxy_pass http://odoo_backend;
    }
}
```
*儲存並退出（`Ctrl + O` -> `Enter` -> `Ctrl + X`）。*

---

### 步驟 5：建立 `docker-compose.yml`

在 `/opt/nginx-odoo/` 目錄建立編排檔：
```bash
sudo nano /opt/nginx-odoo/docker-compose.yml
```

寫入配置：

```yaml
services:
  nginx:
    image: nginx:alpine
    container_name: odoo_nginx_web
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
```

---

### 步驟 6：啟動 Nginx 服務

```bash
cd /opt/nginx-odoo

# 啟動 Nginx 容器
sudo docker compose up -d

# 檢查容器狀態
sudo docker compose ps
```

---

### 步驟 7：設定 Ubuntu UFW 防火牆

對外公開 80 與 443 埠：

```bash
# 開放 HTTP 與 HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 開放 SSH 遠端管理
sudo ufw allow 22/tcp

# 啟用防火牆
sudo ufw enable
sudo ufw reload
sudo ufw status
```

---

### 步驟 8：完整驗證

1. 開啟您的電腦瀏覽器，輸入：
   ```text
   https://<VM_1_的_IP_或網域名稱>
   ```
2. 您應能看到 **Odoo 18 的資料庫建立（Database Manager）畫面**，代表整個三層式鏈路（`Browser` -> `VM 1 Nginx` -> `VM 2 Odoo AP` -> `VM 3 PostgreSQL`）已全數通暢接通！
