---
id: spec.design-system
type: spec
owners: [ui-designer, frontend-dev]
depends_on: [asset.token-css, asset.style-css]
provides: [ui.theme, ui.components, ui.layout-grid]
constraints:
  - "Use Semantic Tokens for colors (e.g., var(--se-color-bg-app-shell))"
  - "Z-index Layering: Bottom Sheet must be 1100"
---

# UI 設計系統規範 (Design Tokens Integration)

## 1. 核心變數 (Tokens)
整合自 `token.css` 與 CSV 定義。

* **Colors**:
    * Brand: `var(--color-brand-500)` (#3B82F6)
    * Status Success: `var(--color-success-500)` (#22C55E)
    * Status Warning: `var(--color-warning-500)` (#F59E0B) - 用於臨時站點
* **Layering (Z-Index)**:
    * `Tabbar`: 100
    * `Sticky`: 150
    * `Overlay`: 1000
    * `Bottom Sheet`: 1100 (最高層級操作)

## 2. 元件對應 (Mapping)
* **App Shell**: 沿用 `style.css` 結構 (`.app-shell`, `.header`).
* **Card**: 重構為 `Status Card`，使用 Grid Layout 排列按鈕。
* **Buttons**:
    * 標準站點: 預設灰色 / 完成綠色 (`.btn-status.done`).
    * 臨時站點: 橘色虛線邊框 (`.btn-status.dynamic`).

## 3. 響應式策略
* **Mobile (<768px)**: Card List View.
* **Desktop (>=768px)**: Data Table View.
