---
id: spec.departure-ops-system
title: 2026 車流退場作業系統 - 技術規格書
version: 2.0.0
last_updated: 2023-10-27
status: Active
type: Technical Specification
owners: [System Architect, Frontend Dev, Backend Dev]
---

# 2026 車流退場作業系統 (2026 Departure Operations System)

本文件定義了「2026 車流退場作業系統」的架構、資料模型、API 介面與前端設計規範。系統旨在協助現場人員透過行動裝置，即時管理車流與人流的退場作業。

## 1. 系統架構 (System Architecture)

本系統採用 **Serverless 架構**，以 Google Ecosystem 為核心：

* **前端 (Frontend)**: Single Page Application (SPA)，Mobile-First 設計。
    * Hosted on: Google Apps Script (HTML Service)
    * Tech Stack: HTML5, CSS3 (Grid/Flexbox), Vanilla JavaScript.
    * Auth: Google Identity Services (GIS).
* **後端 (Backend)**: Google Apps Script (GAS)。
    * API Endpoint: `doGet` (頁面渲染), `doPost` (JSON API).
* **資料庫 (Database)**: Google Sheets。
    * Role: Relational-like database.

---

## 2. 後端規格 (Backend Specification)

### 2.1 模組資訊
* **Module ID**: `module.backend-api`
* **Runtime**: Google Apps Script (V8 Engine)

### 2.2 API 介面 (doPost)
所有請求皆發送至 `doPost`，Payload 為 JSON 格式。

| Action | 描述 | 必要參數 | 回傳 | 權限需求 |
| :--- | :--- | :--- | :--- | :--- |
| `verify` | 驗證 Google Token 並取得權限 | `token` (Google JWT) | `{ok, email, permission}` | Public |
| `getAllData` | 取得所有路線的工作表資料 | `token` | `{ok, allSheets: {Key: {headers, rows}}}` | Viewer+ |
| `getData` | 取得單一工作表資料 | `token`, `sheetName` | `{ok, headers, rows}` | Viewer+ |
| `stamp` | 打卡/更新狀態 (非手動) | `token`, `sheetName`, `carNo`, `colName`, `value` | `{ok, time}` | Leader+ |
| `update` | 手動修正時間/狀態 | `token`, `sheetName`, `carNo`, `colName`, `value` | `{ok, time}` | Leader+ |
| `addColumn` | 新增動態站點欄位 | `token`, `sheetName`, `colName` | `{ok}` | Admin/Leader |

### 2.3 權限模型 (Permission Model)
權限定義於 Google Sheet `人員` 分頁。

* **ADMIN (全功能)**: 全域讀寫、新增欄位。
* **LEADER (退場動線帶領)**: 指定路線讀寫、打卡。
* **VIEWER (檢視)**: 唯讀。
* **None**: 無法使用系統。

### 2.4 資料庫 Schema (Google Sheets)

#### 核心設定表
* **人員 (Auth)**: `[姓名, gmail, 權限]`
* **系統操作紀錄 (Audit Log)**: `[時間, 操作者, 動作, 對象, 欄位, 變更內容]`

#### 業務資料表 (Standard Schema)
系統硬編碼以下標準欄位，後續新增的欄位視為「動態站點」。

1.  **地區專車退場（遊覽車平台）**:
    * Keys: `編號`, `區域車次編號`, `車號`, `車長通知行李到齊`, `Call 車上行李`, `車已上遊覽車平台`, `車前往排車隊`, `車暫停位置`, `退場人員已到齊`, `已 Call 車上遊覽車平台`, `人已前往平台搭車`, `車第二次上遊覽車平台`, `車已離開遊覽車平台`, `備註`
2.  **地區專車退場（禪堂平台）**:
    * Keys: `編號`, `區域車次編號`, `車號`, `車長通知行李到齊`, `Call 車上行李`, `車已上遊覽車平台`, `離開遊覽車平台`, `通過遊覽車平台前往排車隊`, `車停靠位置`, `人已前往禪堂平台搭車`, `車已離開禪堂平台`, `備註`
3.  **水陸專車退場**:
    * Keys: `編號`, `區域車次編號`, `車號`, `確認車已抵達園區`, `車暫停位置`, `退場人員已到齊`, `已 Call 車上遊覽車平台`, `車已上遊覽車平台`, `人從車庫前往平台搭車`, `車已離開遊覽車平台`, `備註`
4.  **288法青退場**:
    * Keys: `編號`, `區域車次編號`, `車號`, `退場人員已到齊`, `車已上遊覽車平台`, `車已離開遊覽車平台`, `備註`

---

## 3. 前端規格 (Frontend Specification)

### 3.1 設計系統 (Design Tokens)
基於 CSS Variables 實作，支援 Mobile-First RWD。

* **色彩 (Colors)**:
    * Primary: `--c-brand` (`#1f6fe5`)
    * Success: `--c-success` (`#22c55e`) / `--bg-success-light` (`#ecfdf5`)
    * Danger: `--c-danger` (`#ef4444`)
    * Text: `--text-main` (`#0f172a`), `--text-sub` (`#64748b`)
* **排版 (Typography)**:
    * System Fonts: `-apple-system`, `BlinkMacSystemFont`, etc.
    * Monospace: 用於時間顯示 (`header-clock`, `action-time`)。
* **佈局 (Layout)**:
    * Max Width: `800px` (App Shell)。
    * Grid System: 用於 Header 與 List Card。

### 3.2 介面組件 (UI Components)

#### A. Header (標題列)
* **Mobile (< 768px)**: 
    * Layout: `44px 1fr 44px` (Grid)。
    * Content: `[Refresh Icon]` - `Time (Centered)` - `[Logout Icon]`。
* **Desktop (>= 768px)**:
    * Layout: Flexbox (`justify-content: space-between`)。
    * Content: `Title` - `Time` - `[Refresh Pills] [Logout Button]`。

#### B. Tab 1: Dashboard (整體情報)
* **Filters**: 橫向捲動按鈕列 (Scrollable Pills)，隱藏 Scrollbar。
* **Stat Cards**: 顯示步驟的完成/未完成數量。
* **Location Cards**: 顯示停車位置統計，支援 Peeking (露頭) 效果 (Mobile 寬度 42%)。

#### C. Tab 2: Operation (登錄與修改)
* **Toolbar**: 包含路線下拉選單與新增站點按鈕。
* **Vehicle Card**: 
    * Layout: 3-Column Grid (`Info` | `Status` | `Button`)。
    * Info: 車號與區域垂直堆疊。
    * Status: 顯示最新標準狀態，若有動態站點則以虛線分隔 (`status-sep`) 顯示於下方。
    * Sorting: 依標準步驟完成度 (Low -> High) 排序。

#### D. Bottom Sheet (控制面板)
* **Trigger**: 點擊列表卡片或地點卡片。
* **Layout**: 
    * Mobile: Action Row 採用垂直堆疊 (Label 上 / Controls 下) 避免按鈕重疊。
    * Desktop: Action Row 採用水平排列。
* **Features**: 
    * 顯示詳細步驟狀態。
    * 執行打卡、略過、修改、刪除。
    * 地點管理 (移除車輛、加入車輛)。

### 3.3 組態設定 (Configuration)
前端 `SHEET_CONFIGS` 定義了 UI 與資料表的對應邏輯：

```javascript
const SHEET_CONFIGS = [
  { 
    key: '地區專車退場（遊覽車平台）', 
    color: '#059669',
    steps: [...], // 定義標準流程步驟
    parkingField: '車暫停位置' // 定義停車欄位名稱
  },
  // ... 其他路線設定
];
