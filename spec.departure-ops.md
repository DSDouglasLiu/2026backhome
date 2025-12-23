---
id: spec.departure-ops
type: spec
owners: [product-owner, system-architect]
depends_on: [gcp.spreadsheet-app, gcp.auth]
provides: [logic.schema-discovery, logic.permission-gate, logic.audit-log]
constraints:
  - "Standard Schema must be hardcoded in GAS to prevent accidental deletion"
  - "Dynamic columns are discovered at runtime via Header inspection"
  - "Write operations must be strictly validated against whitelist permissions (Zero Trust)"
  - "Audit Log must record every state change with actor email"
---

# 2026 車流退場作業系統 - 領域設計規格書

## 1. 領域模型 (Domain Model)

### 1.1 核心實體
* **路線 (Route)**: 對應單一 Google Sheet 分頁。
* **站點 (Station)**: 
    * **標準站點**: 系統硬編碼的固定欄位 (如 "車長通知行李到齊")。
    * **臨時站點 (Dynamic Plugin)**: 現場組長新增的欄位，依附於標準站點之後。
* **權限 (Permission)**:
    * `ADMIN` (全功能): 全域讀寫、新增全域設定。
    * `LEADER` (退場動線帶領): 該路線讀寫、修正、新增站點。
    * `VIEWER` (檢視): 唯讀。

## 2. 資料庫 Schema 定義 (Hardcoded)
以下欄位為系統運作的基準，任何未列於此的後續欄位皆視為「臨時站點」。

* **地區專車退場（遊覽車平台）**: [編號, 區域車次編號, 車號, 車長通知行李到齊, Call 車上行李, 車已上遊覽車平台, 車前往排車隊, 車暫停位置, 退場人員已到齊, 已 Call 車上遊覽車平台, 人已前往平台搭車, 車第二次上遊覽車平台, 車已離開遊覽車平台, 備註]
* **地區專車退場（禪堂平台）**: [編號, 區域車次編號, 車號, 車長通知行李到齊, Call 車上行李, 車已上遊覽車平台, 離開遊覽車平台, 通過遊覽車平台前往排車隊, 車停靠位置, 人已前往禪堂平台搭車, 車已離開禪堂平台, 備註]
* **水陸專車退場**: [編號, 區域車次編號, 車號, 確認車已抵達園區, 車暫停位置, 退場人員已到齊, 已 Call 車上遊覽車平台, 車已上遊覽車平台, 人從車庫前往平台搭車, 車已離開遊覽車平台, 備註]
* **288法青退場**: [編號, 區域車次編號, 車號, 退場人員已到齊, 車已上遊覽車平台, 車已離開遊覽車平台, 備註]

## 3. 安全與稽核
* **寫入閘道**: 每次 `doPost` 必檢查 Email 是否存在於 `人員` 表且具備對應權限。
* **定位策略**: 使用 Column Name 定位寫入，禁止使用 Column Index (防止欄位插入導致錯位)。
* **稽核**: 所有變更寫入 `系統操作紀錄`。
