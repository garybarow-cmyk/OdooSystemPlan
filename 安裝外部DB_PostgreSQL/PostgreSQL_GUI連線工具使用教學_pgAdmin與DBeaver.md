# PostgreSQL GUI 連線工具使用教學（DBeaver 與 pgAdmin 4）

本文件專門指導如何在 Windows 電腦上下載、安裝與設定 **DBeaver** 及 **pgAdmin 4**，連線至您在 Ubuntu VM 上架設的 PostgreSQL 外部資產資料庫。

---

## 目錄
1. [連線前置準備資訊](#1-連線前置準備資訊)
2. [工具一：DBeaver 操作教學（強烈推薦 ⭐⭐⭐⭐⭐）](#2-工具一dbeaver-操作教學強烈推薦-)
   - [2.1 下載與安裝](#21-下載與安裝)
   - [2.2 建立 PostgreSQL 連線步驟](#22-建立-postgresql-連線步驟)
   - [2.3 必學特色功能（JSONB 檢視與 SQL 查詢）](#23-必學特色功能jsonb-檢視與-sql-查詢)
3. [工具二：pgAdmin 4 操作教學（官方工具 ⭐⭐⭐⭐）](#3-工具二pgadmin-4-操作教學官方工具-)
   - [3.1 下載與安裝注意事項](#31-下載與安裝注意事項)
   - [3.2 建立 Server 連線步驟](#32-建立-server-連線步驟)
   - [3.3 常見錯誤排除（含 Access Violation 崩潰問題）](#33-常見錯誤排除含-access-violation-崩潰問題)
4. [DBeaver vs pgAdmin 4 綜合比較](#4-dbeaver-vs-pgadmin-4-綜合比較)

---

## 1. 連線前置準備資訊

在開啟工具連線前，請先確認已備妥以下 Ubuntu VM 的連線參數：

| 參數項目 | 範例數值 | 說明 |
| :--- | :--- | :--- |
| **主機 IP (Host)** | `192.168.1.200` | 您的 Ubuntu VM 實際 IP（可在 VM 內下 `ip a` 查詢） |
| **通訊埠 (Port)** | `5432` | PostgreSQL 預設連接埠 |
| **資料庫名稱 (Database)** | `asset_management_db` | 您在安裝指南建立的資產系統資料庫 |
| **使用者帳號 (User)** | `asset_user` | 資產系統專用使用者（或 `postgres`） |
| **使用者密碼 (Password)** | `您的安全密碼` | 建立帳號時所設定的密碼 |

> ⚠️ **提醒**：請確保 Ubuntu VM 上的 `postgresql.conf` 已將 `listen_addresses` 改為 `'*'`，且 `pg_hba.conf` 及 `ufw` 防火牆已開放您的 Windows 電腦 IP 存取。

---

## 2. 工具一：DBeaver 操作教學（強烈推薦 ⭐⭐⭐⭐⭐）

DBeaver 是一套跨平台的萬用資料庫管理工具，基於 Java JDBC 開發，**穩定性極高、絕不會出現記憶體崩潰，且對 JSONB 資料型態支援非常直覺**。

### 2.1 下載與安裝
1. 前往官網下載免費社群版：[DBeaver Community 官方下載頁面](https://dbeaver.io/download/)
2. 選擇 **Windows Installer** 下載並完成安裝（照預設一路下一步即可）。

---

### 2.2 建立 PostgreSQL 連線步驟

1. **開啟 DBeaver**：
   * 點選左上角帶有 `+` 號的**「新連線」插頭圖示**（或點選功能表 `資料庫` -> `新連線`）。
2. **選擇資料庫類型**：
   * 在清單中選擇 **PostgreSQL**，點擊「下一步」。
3. **填寫連線參數 (Main 分頁)**：
   * **Host**：輸入 Ubuntu VM 的 IP（例如 `192.168.1.200`）。
   * **Port**：`5432`。
   * **Database**：`asset_management_db`。
   * **Username**：`asset_user`。
   * **Password**：輸入您的資料庫密碼。
   * 勾選「儲存密碼」（Save password locally）。
4. **下載驅動程式 (Driver)**：
   * 首次連線時，點擊視窗左下角的 **「測試連線 (Test Connection)」**。
   * DBeaver 會彈出視窗詢問是否下載 PostgreSQL JDBC 驅動程式，直接點選 **「下載 (Download)」**（約數秒完成）。
5. **確認連線成功**：
   * 測試完成後會跳出 `Connected`（伺服器版本、驅動版本等資訊）。
   * 點擊右下角 **「完成 (Finish)」** 即可儲存連線。

---

### 2.3 必學特色功能（JSONB 檢視與 SQL 查詢）

* **瀏覽資料表**：
  * 在左側導覽樹依序展開：`連線名稱` -> `Databases` -> `asset_management_db` -> `Schemas` -> `public` -> `Tables`。
  * 雙擊 `assets` 資料表，點選右側上方切換至 **「資料 (Data)」** 分頁即可直接查看表格內容。
* **檢視/修改 JSONB 規格欄位（超好用）**：
  * 資產的 `specifications` 欄位是 JSONB。在 DBeaver 中對著該欄位點一下，右側面板會自動切換為 **JSON 格式化視圖**（具備縮排與色彩標記），點擊即可直接修改。
* **開啟 SQL 查詢視窗**：
  * 按快捷鍵 **`Ctrl + ]`**（或點選上方工具列的「SQL」圖示），即可直接輸入 SQL 指令查詢，按 `Ctrl + Enter` 即可執行單行或選取的 SQL。
* **產生 ER 關聯圖**：
  * 點擊資料庫或 Schema 的 **「ER 圖 (ER Diagram)」** 分頁，自動繪製出資料表外鍵關聯圖。

---

## 3. 工具二：pgAdmin 4 操作教學（官方工具 ⭐⭐⭐⭐）

pgAdmin 4 是 PostgreSQL 官方維護的管理工具，採用網頁/桌面複合架構，功能全面，但初次設定較嚴謹。

### 3.1 下載與安裝注意事項
* 請務必確認下載的是 **[pgAdmin 4](https://www.pgadmin.org/download/pgadmin-4-windows/)**（切勿使用舊版 pgAdmin 3）。
* 安裝後第一次啟動時，pgAdmin 會要求您設定一組 **主密碼 (Master Password)**，此密碼用於保護本機儲存的連線資訊，請務必記住。

---

### 3.2 建立 Server 連線步驟

1. **進入連線新增畫面**：
   * 在左側的物件樹狀目錄中，對著 **`Servers`** 點擊滑鼠右鍵。
   * 選擇 **`Register`** -> **`Server...`**。
2. **General (一般) 分頁**：
   * **Name**：自行輸入辨識名稱（例如 `Ubuntu-資產外部DB`）。
3. **Connection (連線) 分頁**（關鍵設定）：
   * **Host name/address**：輸入 Ubuntu VM 的 IP（例如 `192.168.1.200`）。
   * **Port**：`5432`。
   * **Maintenance database**：輸入 `asset_management_db`（或是預設 `postgres`）。
   * **Username**：`asset_user`。
   * **Password**：輸入您的密碼。
   * **Save password?**：可選擇勾選儲存。
4. **點擊右下角「Save」儲存**：
   * 儲存後，左側清單會出現該伺服器節點，點開後若能展開各個資料庫即表示連線成功！

---

### 3.3 常見錯誤排除（含 Access Violation 崩潰問題）

#### 問題 1：彈出 `Access violation writing 0x0000000000000000` 崩潰
* **原因**：Windows 憑證管理員與 pgAdmin 內部 Python/加密金鑰衝突，或使用了舊版軟體。
* **解法**：
  1. 在建立連線時，**取消勾選「Save password?」**，改為每次連線時彈出視窗手動輸入。
  2. 關閉 pgAdmin，按 `Win + R` 輸入 `%APPDATA%`，將 `pgadmin` 資料夾重新命名為 `pgadmin_old` 後重開工具重設。

#### 問題 2：出現 `Connection refused` 或 `Timeout expired`
* **檢查項目**：
  1. 請檢查 Ubuntu VM 終端機內是否已執行 `sudo systemctl restart postgresql`。
  2. 檢查 Ubuntu VM 防火牆：`sudo ufw status` 是否已放行 5432 埠。
  3. 檢查 `/etc/postgresql/16/main/pg_hba.conf`，是否有加入 Windows 電腦 IP 的存取允許行：
     ```conf
     host    asset_management_db    asset_user    0.0.0.0/0    scram-sha-256
     ```

#### 問題 3：密碼驗證失敗 `FATAL: password authentication failed for user "asset_user"`
* **解法**：
  * 在 Ubuntu VM 終端機內重設該帳號密碼：
    ```bash
    sudo -u postgres psql -c "ALTER USER asset_user WITH PASSWORD '新密碼';"
    ```

---

## 4. DBeaver vs pgAdmin 4 綜合比較

| 評比項目 | DBeaver Community | pgAdmin 4 |
| :--- | :--- | :--- |
| **軟體核心架構** | Java / Eclipse 核心（原生桌面） | Python / NW.js (類網頁型) |
| **連線穩定性** | ⭐⭐⭐⭐⭐（非常穩定，無記憶體崩潰） | ⭐⭐⭐（偶有前端或主密碼崩潰） |
| **啟動速度** | 較快 | 較慢（需在背景啟動 web 服務） |
| **JSONB 檢視與編輯** | **極佳**（獨立 JSON 面板、語法高亮、摺疊） | 普通（純文字檢視） |
| **多資料庫支援** | 支援 PostgreSQL、MySQL、SQL Server、Oracle 等 | 僅專門支援 PostgreSQL / EDB |
| **視覺化監控儀表板** | 較陽春 | **較豐富**（內建連線 Session、交易量圖表） |
| **新手上手難易度** | **簡單直覺** | 稍微繁複 |

> **建議**：
> * 平常進行**資料表管理、SQL 測試、檢視資產規格 JSON 內容**，強烈推薦使用 **DBeaver**。
> * 若要觀察 PostgreSQL 伺服器的效能曲線圖表與 Session 狀態，可以使用 **pgAdmin 4**。
