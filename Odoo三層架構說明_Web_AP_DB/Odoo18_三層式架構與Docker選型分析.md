# Odoo 18 三層式架構設計與 Docker 容器化評估指南

本文件針對企業級 **Odoo 18** 部署，詳細解析 **Web（反向代理層）+ AP（應用服務層）+ DB（資料庫層）** 的三層式（Three-Tier）架構規劃，並針對 **3 台獨立 Ubuntu VM** 的環境，深入評估**是否需要導入 Docker 架構**。

---

## 目錄
1. [三層式架構全景圖](#1-三層式架構全景圖)
2. [各層核心職責與配置重點](#2-各層核心職責與配置重點)
   - [2.1 Odoo Web (反向代理與安全層)](#21-odoo-web-反向代理與安全層)
   - [2.2 Odoo AP (應用與商業邏輯層)](#22-odoo-ap-應用與商業邏輯層)
   - [2.3 Odoo DB (核心資料庫層)](#23-odoo-db-核心資料庫層)
3. [核心評估：3 台 Ubuntu 需要使用 Docker 嗎？](#3-核心評估3-台-ubuntu-需要使用-docker-嗎)
   - [3.1 Docker vs 原生安裝 (Native) 綜合比較](#31-docker-vs-原生安裝-native-綜合比較)
   - [3.2 各層的最佳落地建議（混合架構最推薦）](#32-各層的最佳落地建議混合架構最推薦)
4. [3 台 Ubuntu VM 建議硬體與網路規格表](#4-3-台-ubuntu-vm-建議硬體與網路規格表)
5. [實務部署規劃與通訊矩陣](#5-實務部署規劃與通訊矩陣)

---

## 1. 三層式架構全景圖

三層分離架構是企業級 ERP 系統最標準、最具彈性與安全性的一種拓撲：

```mermaid
graph TD
    User["外部使用者 / 瀏覽器"] -->|HTTPS :443| Web["VM 1: Odoo Web (Nginx)<br>反向代理 / SSL終端 / 靜態快取 / WAF"]
    
    subgraph "應用與商業邏輯層"
        Web -->|HTTP :8069 (業務操作)| AP["VM 2: Odoo 18 AP (Application)<br>Python 3.12+ / Werkzeug / ORM<br>報表渲染 (wkhtmltopdf) / 排程任務"]
        Web -->|HTTP :8072 (即時長輪詢/Websocket)| AP
        AP -->|本機/NAS掛載| Filestore["附件與圖片儲存 (Filestore)"]
    end
    
    subgraph "資料庫儲存層"
        AP -->|Port 5432 (主庫連線)| DB["VM 3: Odoo DB (PostgreSQL 16)<br>核心 ERP 資料庫 (交易、帳務、庫存)"]
        AP -.->|Port 5432 (模組跨庫串接)| ExtDB["外部獨立 DB (資產管理系統)<br>存放設備動態規格 (JSONB)"]
    end
```

---

## 2. 各層核心職責與配置重點

### 2.1 Odoo Web (反向代理與安全層)
* **核心軟體**：Nginx（最推薦）或 Traefik。
* **主要任務**：
  1. **SSL/TLS 終端解密**：統一在此層管理 HTTPS 憑證（如 Let's Encrypt 自動續簽），後方 AP 伺服器僅跑 HTTP，降低 AP 運算負擔。
  2. **靜態資源快取**：快取 Odoo 的 CSS、JavaScript、圖片等前端靜態檔案（`/web/static/*`），直接回應瀏覽器，節省 AP 流量與 CPU。
  3. **請求分流 (Routing)**：
     * 一般請求導向 AP 的 `8069` 埠。
     * 即時通訊與聊天（Websocket / Longpolling）導向 AP 的 `8072` 埠。
  4. **安全性與負載保護**：阻擋惡意攻擊、設定 IP 白名單、限制上傳檔案大小 (`client_max_body_size 100M`)，防止慢速連線佔用 AP Worker。

### 2.2 Odoo AP (應用與商業邏輯層)
* **核心軟體**：Odoo 18 官方程式庫、Python 3.12+、Wkhtmltopdf（PDF 報表產生器）。
* **主要任務**：
  1. **商業邏輯運算**：執行進銷存、財務、製造、HR 等所有業務邏輯與客製化模組。
  2. **行程併發處理 (Multiprocessing)**：
     * 配置 `workers = (CPU核心數 * 2) + 1`，啟用多行程 Worker 提高大量並發操作時的穩定度。
  3. **排程任務 (Cron Jobs)**：自動檢查逾期款項、庫存補貨、定期與外部系統資料同步。
  4. **資料庫連線池 (Connection Pool)**：集中向 VM 3（Odoo DB）與外部資產 DB 發送 SQL 查詢請求。

### 2.3 Odoo DB (核心資料庫層)
* **核心軟體**：PostgreSQL 16 / 17。
* **主要任務**：
  1. **專注於高效 I/O 與交易事務**：這台主機**完全不運行任何 Web 或 Python 應用**，將所有記憶體（RAM）與 CPU 計算資源百分之百留給資料庫快取（Buffer Cache）與索引搜尋。
  2. **硬碟寫入與 WAL 記錄**：處理 Odoo 龐大的 ACID 交易日誌（Write-Ahead Logging）。
  3. **資料庫安全防護**：僅對 VM 2（Odoo AP）開放 5432 埠，完全不對外網開放，杜絕直接被掃描攻擊的風險。

---

## 3. 核心評估：3 台 Ubuntu 需要使用 Docker 架構嗎？

**結論簡述**：
> **不是「非用不可」，但強烈推薦在「Web 層」與「AP 層」使用 Docker 架構；「DB 層」則可視團隊維運習慣選擇 Docker 或原生安裝。**

### 3.1 Docker vs 原生安裝 (Native) 綜合比較

| 評估維度 | 原生安裝 (Native apt/systemd) | Docker 容器化架構 | 實務分析與結論 |
| :--- | :--- | :--- | :--- |
| **部署複雜度** | 需要手動安裝幾十個 Python 函式庫、wkhtmltopdf、設定虛擬環境 (`venv`) | 撰寫一份 `docker-compose.yml`，一鍵 `docker compose up -d` 即可開機 | **Docker 大勝**：大幅減少環境配置錯誤 |
| **Python 環境依賴衝突** | Ubuntu 24/26 啟用了 **PEP 668**（禁止全域 pip install），易與系統 Python 衝突 | 容器完全封裝，自帶完整 Python 執行環境，與 Host OS 完全隔離 | **Docker 大勝**：避免套件升級時導致系統癱瘓 |
| **模組開發與更新** | 更新模組需重啟系統服務，若 Python 套件版本升級失敗難以回退 | 自訂模組直接用 Volume 掛載；要測試新版本隨時切換 Image Tag | **Docker 勝**：升級與熱插拔極為方便 |
| **效能損耗 (Performance)** | 原生無中介層，效能 100% | Linux 上 Docker 是透過 cgroups/namespace 運作，**CPU/記憶體損耗接近 0%** | **平手**（Linux 下容器幾無額外負擔） |
| **資料持久化 (Persistence)** | 資料夾路徑固定直覺 | 必須嚴格設定 Volume 掛載，忘記掛載可能導致重啟容器資料遺失 | **原生較直覺**：但只要熟悉 Volume 即可規避 |
| **備份與遷移** | 需手動打包檔案與資料庫 | 將 Compose 檔與掛載目錄壓縮，直接複製到新主機 1 分鐘開機接手 | **Docker 大勝**：災難復原能力極高 |

---

### 3.2 各層的最佳落地建議（推薦：混合式架構）

針對您的 3 台獨立 Ubuntu 26 VM，最推薦的黃金架構是：

```
[VM 1: Web]    --> Docker 跑 Nginx + Certbot（輕量、換憑證超快）
[VM 2: AP]     --> Docker 跑 Odoo 18（環境乾淨、自訂模組易擴展、升級零風險）
[VM 3: DB]     --> 原生安裝 PostgreSQL 16（硬碟 I/O 調優直覺、DBA 維護最習慣）
```

#### 為什麼 VM 2 (Odoo AP) 特別強烈建議用 Docker？
1. **依賴套件過於繁複**：Odoo 依賴超過 40 個 Python 函式庫，還有專屬的 C 編譯庫（如 `libxml2`、`libxslt`、`openldap`），自己手動在 Ubuntu 上安裝與除錯往往需要花費數小時。
2. **擴展容易 (Scalability)**：未來如果公司人數增加、系統負載變大，只要在 Docker 內調整 Worker 數量，或者直接新增一台 VM 跑同樣的 Docker 映像檔，立刻完成 AP 橫向負載平衡。
3. **自訂模組開發極為友善**：在 VM 上開一個 `addons/` 目錄放置您開發的「資產管理串接模組」，直接掛載進容器，修改程式後 `docker compose restart web` 即套用。

#### 為什麼 VM 3 (PostgreSQL) 可以不用 Docker？
* 資料庫核心通常講求最純粹的**磁碟 I/O 控制、系統層級的檔案系統（如 ext4 / XFS）監控、以及直接使用 Linux 實體記憶體的快取機制**。
* 原生安裝讓您在下達 `psql`、`pg_dump` 或是調校 `/etc/postgresql/16/main/postgresql.conf` 時最直觀，不必經過 `docker exec`。
*(註：若您全團隊皆已熟悉 Docker，將 DB 容器化並掛載實體 Volume 亦完全可行)*

---

## 4. 3 台 Ubuntu VM 建議硬體與網路規格表

| 虛擬機角色 | 主機名稱範例 | 建議規格 | 建議作業系統 | 建議安裝方式 |
| :--- | :--- | :--- | :--- | :--- |
| **Web 伺服器** | `odoo-web` | 2 vCPU / 2GB RAM / 30GB SSD | Ubuntu 26.04 LTS | **Docker** (Nginx 容器) |
| **AP 伺服器** | `odoo-app` | 4 vCPU / 8GB RAM / 60GB SSD | Ubuntu 26.04 LTS | **Docker** (Odoo 18 容器) |
| **DB 伺服器** | `odoo-db` | 4 vCPU / 8GB ~ 16GB RAM / 100GB+ 高速 SSD | Ubuntu 26.04 LTS | **原生安裝** (PostgreSQL 16) |

---

## 5. 實務部署規劃與通訊矩陣

為了確保系統安全性，3 台 VM 之間的防火牆放行原則如下：

```
[外部使用者] 
     │ (僅開放 80, 443)
     ▼
┌───────────────────────────────┐
│ VM 1: Odoo Web (Nginx)        │
└──────────────┬────────────────┘
               │ (僅對 VM1 開放 8069, 8072)
               ▼
┌───────────────────────────────┐
│ VM 2: Odoo AP (Odoo 18)       │
└───────┬───────────────┬───────┘
        │               │
        │ (僅開放 5432)  │ (僅開放 5432)
        ▼               ▼
┌──────────────┐ ┌──────────────────────┐
│ VM 3: Odoo DB│ │ 外部資產管理 DB (VM) │
└──────────────┘ └──────────────────────┘
```

* **VM 1 (Web)**：對外公開 **80 (HTTP)** 與 **443 (HTTPS)**。
* **VM 2 (AP)**：本機關閉公網直連，僅允許來自 **VM 1 的 IP** 存取 `8069` 與 `8072` 埠。
* **VM 3 (DB)**：本機關閉公網直連，僅允許來自 **VM 2 的 IP** 存取 `5432` 埠。
* **外部資產 DB**：僅允許來自 **VM 2 的 IP** 存取 `5432` 埠（由 AP 上的自訂模組進行資料交換）。
