---
id: spec.departure-ops
type: spec
owners: [product-owner, system-architect]
depends_on: [gcp.spreadsheet-app, gcp.auth, frontend.google-identity-services]
provides: [logic.schema-discovery, logic.permission-gate, logic.audit-log, ui.mobile-first-spa]
version: 2.0.0
last_updated: 2025-12-25
constraints:
  - "Standard Schema must be hardcoded in GAS to prevent accidental deletion"
  - "Dynamic columns are discovered at runtime via Header inspection"
  - "Write operations must be strictly validated against whitelist permissions (Zero Trust)"
  - "Audit Log must record every state change with actor email"
  - "Frontend must strictly follow Mobile-First principles with responsive adaptations for Desktop"
---

# 2026 車流退場作業系統 - 系統規格書

## 1. 系統架構概觀 (System Overview)

本系統為 **Mobile-First (手機優先)** 的單頁應用程式 (SPA)，旨在協助現場人員管理車流與人流退場作業。系統設計強調在移動情境下的操作效率與資訊可讀性。

### 1.1 技術棧 (Tech Stack)
* **前端 (Frontend)**: 原生 HTML5, CSS3 (CSS Grid/Flexbox), JavaScript (ES6+)。無依賴龐大框架，確保輕量化載入。
* **認證 (Auth)**: Google Identity Services (GIS) - 透過 Google Sign-In 取得 ID Token。
* **後端 (Backend)**: Google Apps Script (GAS) `doPost`/`doGet` 接口，作為 Serverless API。
* **資料庫 (Database)**: Google Sheets，作為關聯式資料儲存與視覺化後台。

---

## 2. 領域模型 (Domain Model)

### 2.1 核心實體
* **路線 (Route)**: 對應單一 Google Sheet 分頁（例如：「地區專車退場（遊覽車平台）」）。
* **站點 (Station)**: 
    * **標準站點**: 系統硬編碼 (Hardcoded) 的固定欄位，對應標準作業程序 (SOP)。
    * **臨時站點 (Dynamic Plugin)**: 現場組長動態新增的欄位，系統於執行時自動偵測並依附於標準站點之後。
* **權限 (Permission)**:
    * `ADMIN` (全功能): 全域讀寫、新增全域設定。
    * `LEADER` (退場動線帶領): 該路線讀寫、修正、新增站點。
    * `VIEWER` (檢視): 唯讀權限。

### 2.2 資料庫 Schema 定義 (Hardcoded)
以下欄位為系統運作的基準，任何未列於此的後續欄位皆視為「臨時站點」。

* **地區專車退場（遊覽車平台）**: 
    `[編號, 區域車次編號, 車號, 車長通知行李到齊, Call 車上行李, 車已上遊覽車平台, 車前往排車隊, 車暫停位置, 退場人員已到齊, 已 Call 車上遊覽車平台, 人已前往平台搭車, 車第二次上遊覽車平台, 車已離開遊覽車平台, 備註]`
* **地區專車退場（禪堂平台）**: 
    `[編號, 區域車次編號, 車號, 車長通知行李到齊, Call 車上行李, 車已上遊覽車平台, 離開遊覽車平台, 通過遊覽車平台前往排車隊, 車停靠位置, 人已前往禪堂平台搭車, 車已離開禪堂平台, 備註]`
* **水陸專車退場**: 
    `[編號, 區域車次編號, 車號, 確認車已抵達園區, 車暫停位置, 退場人員已到齊, 已 Call 車上遊覽車平台, 車已上遊覽車平台, 人從車庫前往平台搭車, 車已離開遊覽車平台, 備註]`
* **288法青退場**: 
    `[編號, 區域車次編號, 車號, 退場人員已到齊, 車已上遊覽車平台, 車已離開遊覽車平台, 備註]`

### 2.3 安全與稽核
* **寫入閘道 (Write Gateway)**: 每次 `doPost` 請求必檢查 Token 的有效性，並驗證 Email 是否存在於 `人員` 表且具備對應權限。
* **定位策略**: 使用 Column Name 動態搜尋 Column Index 進行寫入，禁止使用固定 Index (防止欄位插入導致錯位)。
* **稽核 (Audit)**: 所有變更 (STAMP, UPDATE, ADD_COL) 均寫入 `系統操作紀錄` 分頁。

---

## 3. 前端規格 (Frontend Specifications)

### 3.1 設計規範 (Design Tokens)
採用 CSS Variables 定義全域樣式，確保一致性。

* **色彩系統**:
    * Primary: `--c-brand: #1f6fe5` (主色藍)
    * Semantic: `--c-success: #22c55e` (綠), `--c-danger: #ef4444` (紅), `--c-neutral: #64748b` (灰)
    * Backgrounds: `--bg-body: #e5e7eb`, `--bg-surface: #ffffff`
* **排版**:
    * 字體: 系統預設黑體 (`-apple-system`, `Roboto`, etc.)。
    * 數據顯示: 強制使用 `Monospace` 等寬字體，確保時間與數字對齊。

### 3.2 響應式佈局策略 (RWD Strategy)

系統針對 **手機 (< 768px)** 與 **電腦 (>= 768px)** 提供差異化的介面體驗。

#### A. 標題列 (Header)
* **手機版**: 
    * 採用 **三欄式 Icon 佈局**：`[更新按鈕] [時間] [登出按鈕]`。
    * 時間置中顯示，移除標題文字以釋放空間。
    * 隱藏更新頻率選單，預設固定更新頻率。
* **電腦版**: 
    * 還原完整資訊：`[標題] [時間] [功能區]`。
    * 顯示完整的「更新頻率」切換按鈕 (5分/2分/60秒/30秒) 與文字版登出按鈕。

#### B. Tab 1: 整體情報 (Dashboard)
* **篩選列 (Filters)**: 採用 **Scrollable Pills** 設計，隱藏 Scrollbar (`scrollbar-width: none`)，保留滑動慣性。
* **地點卡片 (Location Cards)**: 
    * 手機版採用 **Peeking (露頭)** 設計，卡片寬度設為 42%，讓使用者直覺感知可橫向滑動。
    * 電腦版自動轉為 Grid 網格排列。

#### C. Tab 2: 登錄與修改 (Operations)
* **列表卡片 (List Cards)**:
    * 採用 **CSS Grid 三欄式垂直堆疊** 佈局：
        1.  **資訊區 (左)**: 車號與區域垂直排列。
        2.  **狀態區 (中)**: 
            * 標準流程狀態與時間。
            * 若有「臨時站點」打卡，透過虛線分隔 (`status-sep`) 於下方列出。
            * 使用 `minmax(0, 1fr)` 防止長文字擠壓按鈕。
        3.  **操作區 (右)**: 展開按鈕。
* **控制面板 (Bottom Sheet)**:
    * 點擊卡片後由下方滑出。
    * 手機版內容強制 **雙層堆疊** (`Label` 上 / `Time + Buttons` 下)，避免按鈕重疊。
    * **地點管理**: 針對地點卡片點擊後，提供「移除現有車輛」與「加入新車輛」的管理介面。

### 3.3 核心功能模組

#### 1. 自動更新機制
* **Polling**: 使用 `setInterval` 定期呼叫後端 API。
* **頻率**:
    * 手機版: 預設 **20秒 (`MOBILE_REFRESH_MS`)**，不提供切換以簡化介面。
    * 電腦版: 預設 **60秒**，提供 5分/2分/60秒/30秒 切換選項。
* **手動更新**: 提供獨立的 Refresh Icon，點擊後立即更新並重置計時器。

#### 2. 狀態顯示邏輯
* **排序**: 依據「標準步驟完成度」由低到高排序，將未完成的車輛置頂。
* **多行狀態**: 
    1. 優先顯示最新的「標準站點」狀態。
    2. 比對時間，若有「臨時站點」的時間晚於標準站點，則依序顯示於下方。

#### 3. 身份驗證
* 前端整合 `g_id_onload` 與 `g_id_signin`。
* 取得 JWT 後傳送至後端 `verify` 接口。
* 後端回傳使用者權限 (`ADMIN`/`LEADER`/`VIEWER`)，前端據此控制 UI 元素 (如「新增站點」按鈕的可見性)。
