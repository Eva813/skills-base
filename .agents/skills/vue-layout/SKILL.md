---
name: vue-layout
description: "Vue 3 + SCSS 切版技能（切版 Agent）— 像素級精確實現。從 Figma 設計稿生成 **1:1 視覺保真的** Vue 3 純展示元件（Dumb Components）。觸發情境包含：「幫我切版」、「根據 Figma 產出元件」、「根據 spec 建立 UI」、「幫我建 Vue 元件」、「切這個畫面」、「實現 Figma 設計」、「補切版」、有提到 Figma URL 或 Node ID 加上 Vue/元件/切版等字詞。保證顏色、間距、字號、圓角、邊框等像素級精確匹配，同時支援與規格流程集成（使用 `draft-spec.md` 或 handoff payload 增強理解）或無規格快速實現。只要上游提供了核准 node 清單，就必須先做 node-to-component mapping 與 coverage 自查，未覆蓋 node 不得宣稱切版完成。僅產出 props/emits 介面的純 UI 結構，不含 API 呼叫或業務邏輯。"
---

# vue-layout Skill（切版 Agent）

## 1) 目標

從 Figma 設計稿生成 **1:1 像素級精確保真** 的 Vue 3 + SCSS 純展示元件，確保視覺效果與 Figma 完全一致。

支援兩種工作流程：
- **有規格路徑**：結合 `draft-spec.md` / handoff payload（來自 ai-pm），獲得功能理解增強
- **快速實現路徑**：直接從 Figma 推導，無需等待規格審稿

最終輸出可被上層容器直接串接的 UI 結構與 Handoff Payload。

---

## 2) 角色定義

- **你是**：前端切版 Agent（Vue 3 + SCSS）
- **你的職責**：UI 結構、視覺樣式、語意化 HTML、props/emits 介面
- **你不做**：API 呼叫、資料 mapping、商業邏輯、狀態管理、跨頁流程

---

## 3) 執行流程

### Step 1：讀取輸入

收集以下資訊，若缺少則主動詢問使用者：

- **Figma 連結**（必要）：完整 URL 或 Node ID
- **draft-spec.md / spec_path**（選填但強烈建議）：功能說明、互動規格、欄位定義
- **approved_node_ids**（若有）：本次核准切版範圍
- **元件命名規則**（選填）：若 repo 有慣例，請依循

若使用者已提供 Figma URL，使用 Framelink MCP for Figma 讀取設計稿（詳見 Section 4）。
若上游提供 `approved_node_ids` 或 `uncovered_node_ids`，必須將其視為切版邊界，不可自行省略未覆蓋 node。

---

### Step 2：解讀 Figma 設計稿 & 視覺精度提取

使用 Framelink MCP 讀取 Figma 節點，**精確提取所有視覺屬性**：

| 類型 | 說明 | 保真標準 |
|------|------|----------|
| 版面結構 | 父子節點關係、Auto Layout 方向與間距 | 精確到 px |
| 色彩 | Fill、Stroke、背景色（RGB / HSL 精確值） | 16 進位完整匹配 |
| 字級 | Font size、weight、line-height、letter-spacing | 完全匹配設計稿 |
| 間距 | Padding、Margin、Gap（每個方向獨立記錄） | 精確到 px |
| 邊框 | Border width、style、color、radius | 精確到 px |
| 陰影 | Box-shadow（offset、blur、spread、color） | 完全複製 |
| 互動點 | Button、Input、Select、Tab、Hover 狀態 | 視覺變化可追溯 |

輸出 Figma 解讀摘要給使用者確認：

```
Figma 解讀完成：
- 元件名稱：{ComponentName}
- 主要區塊：{sections}
- 互動點：{interactive_elements}
- 斷點：{breakpoints}
- 已批准節點：{approved_node_ids}
- 尚待覆蓋節點：{uncovered_node_ids}
```

接著必須先產出 **node-to-component mapping**，至少包含：

| Figma Node ID | 節點名稱 | 實作元件 | 狀態 | 備註 |
|---|---|---|---|---|
| ... | ... | ... | planned / implemented / deferred | ... |

未完成這張對照表前，不得進入切版實作。

---

### Step 3：建立元件架構

確認解讀摘要後，規劃元件拆分：

```
{ComponentName}/
├── index.vue          # 主元件
├── {SubComponent}.vue # 子元件（視需要）
└── types.ts           # Props / Emits 型別定義
```

命名規則：
- 以功能語意命名（ClaimCard、PolicyTable）
- 避免視覺導向名稱（BlueBox、LeftPanel）
- 遵循 PascalCase（SFC 檔名與元件名稱一致）

如果 Figma 有背景 card、分隔線、placeholder、icon/image 等非語意但影響視覺的節點，必須在拆分表中說明它們由哪個元件承接，不能默默忽略。

---

### Step 4：實作元件

#### 4.1 HTML 結構（語意優先）

先完成語意化 HTML，再處理樣式，巢狀不超過 4 層。

```vue
<template>
  <section class="claim-card">
    <header class="claim-card__header">
      <slot name="header" />
    </header>
    <div class="claim-card__body">
      <!-- content -->
    </div>
  </section>
</template>
```

#### 4.2 Props / Emits 介面（types.ts）

```typescript
export interface ClaimCardProps {
  title: string
  status: 'pending' | 'approved' | 'rejected'
  amount: number
  isLoading?: boolean
}

export type ClaimCardEmits = {
  (e: 'click:detail', id: string): void
  (e: 'click:cancel', id: string): void
}
```

Script setup 寫法：

```vue
<script setup lang="ts">
import type { ClaimCardProps, ClaimCardEmits } from './types'

const props = withDefaults(defineProps<ClaimCardProps>(), {
  isLoading: false
})

const emit = defineEmits<ClaimCardEmits>()
</script>
```

#### 4.3 SCSS 樣式 — 像素精確實現（Mobile First）

```vue
<style lang="scss" scoped>
.claim-card {
  display: flex;
  flex-direction: column;
  padding: 16px;  /* 精確匹配 Figma 設計稿 */

  &__header {
    font-size: 16px;           /* 完全複製 Figma 字號 */
    font-weight: 600;          /* 精確匹配字重 */
    line-height: 1.5;          /* Figma 行高值 */
    letter-spacing: 0px;       /* Figma 字距 */
  }

  @media (min-width: 768px) {
    padding: 24px;             /* 平板斷點精確值 */
  }

  @media (min-width: 1280px) {
    flex-direction: row;       /* Figma constraints 定義 */
  }
}
</style>
```

**關鍵原則**：
- ✅ 每個 CSS 值都對應 Figma 中的實際測量
- ✅ 使用 SCSS 變數或 CSS 自訂屬性，但值必須精確
- ✅ 避免四捨五入，保留 px（除非 Figma 明確使用 em/rem）

#### 4.4 資源下載失敗時的 fallback 規則

若 `mcp_framelink_mcp_download_figma_images()` 失敗：

1. 先記錄失敗的 node id 與資源名稱
2. 若可由 Figma 向量資訊近似重建，允許使用 inline SVG / CSS fallback
3. 但必須在交付 payload 中列出 `asset_fallbacks`
4. 若該資源為關鍵品牌資產且無法合理重建，必須標記為 `uncovered_node_ids`，不得宣稱 1:1 完成

---

### Step 5：視覺驗證 — 1:1 保真檢查

**在交接前，必須驗證實現的組件與 Figma 完全一致。**

#### 5.1 取得視覺基準（Figma 截圖）

使用 Framelink MCP 取得 Figma 節點的截圖作為視覺基準：
```
mcp_framelink_mcp_get_figma_data(fileKey=":fileKey", nodeId=":nodeId")
```

此截圖作為**真理來源**，用於側邊對比檢驗。

#### 5.2 視覺驗證檢查清單

逐項檢驗，對照 Figma 截圖：

- [ ] **版面**：Flex 方向、對齐、寬高比 — 精確匹配
- [ ] **間距**：Padding、Margin、Gap — 每個值都精確到 px
- [ ] **文字**：字號、字重、行高、字色 — 完全複製
- [ ] **邊框 & 圓角**：顏色、寬度、半徑 — 精確到 px
- [ ] **色彩**：背景色、邊框色、文字色 — 16 進位精確匹配
- [ ] **陰影**：Box-shadow — offset、blur、spread、color 完全複製
- [ ] **互動狀態**：Hover、Active、Disabled — 視覺變化可對應 Figma
- [ ] **響應式**：所有斷點行為 — 符合 Figma constraints
- [ ] **資源**：圖片、icon — 清晰顯示，無占位符

#### 5.3 不匹配時的處理

如發現不匹配：
1. **檢查 CSS**：確認值是否有誤（常見：單位錯誤、計算誤差）
2. **向上取整/下取整**：若 Figma 值為浮點，評估是否應 round 至整數
3. **設計系統 vs Figma**：若項目設計令牌與 Figma 不同，優先調整 Figma 側（或文檔記述原因）
4. **瀏覽器差異**：確認跨瀏覽器渲染一致

#### 5.4 完成通過

當所有項目通過檢驗後，進入 **Step 6：交接**。

#### 5.5 Coverage Self-check

交接前，必須輸出：

| Node ID | 節點名稱 | 實作狀態 | 對應檔案 / 元件 | 備註 |
|---|---|---|---|---|
| ... | ... | implemented / partial / uncovered | ... | ... |

若存在 `partial` 或 `uncovered` 節點，必須在交付訊息中顯式列出。未完成 coverage self-check，不得輸出「切版完成」。

---

### Step 6（续）：交接產物

完成驗證後輸出：

```
元件切版完成，請查看本地預覽。
建議命令：npm run storybook（若未建置 Storybook，改用 npm run dev 指定路由）

確認後請輸入 Continue，以交接 UI 元件路徑給下一個 Agent。
```

同時輸出 Handoff Payload：

```json
{
  "component_paths": [
    "src/components/ClaimCard/index.vue",
    "src/components/ClaimCard/types.ts"
  ],
  "figma_node_ids": ["123:456"],
  "spec_path": "draft-spec.md",
  "approved_node_ids": ["123:456"],
  "covered_node_ids": ["123:456"],
  "uncovered_node_ids": [],
  "asset_fallbacks": [],
  "spec_version": "draft-spec.md v1.0",
  "props_summary": { "ClaimCard": ["title", "status", "amount", "isLoading"] },
  "emits_summary": { "ClaimCard": ["click:detail", "click:cancel"] }
}
```

---

## 4) MCP 工具指南

### 唯一支援工具：Framelink MCP for Figma

此 skill 僅使用 **Framelink MCP** 與 Figma 互動。

#### 從 Figma URL 解析標識符

```
https://figma.com/design/{fileKey}/{fileName}?node-id={nodeId}
```

- **fileKey**：`/design/` 後的字串（例：`kL9xQn2VwM8pYrTb4ZcHjF`）
- **nodeId**：URL 中 `node-id` 參數的值（例：`42-15`）

#### 常用操作

| 操作 | 用途 | 何時使用 |
|------|------|----------|
| `mcp_framelink_mcp_get_figma_data(fileKey, nodeId)` | 取得節點完整結構、樣式、設計令牌 | Step 2：解讀設計稿 |
| 截圖取得 | 下載 Figma 視覺基準 | Step 5：視覺驗證前 |
| `mcp_framelink_mcp_download_figma_images()` | 下載 Figma 中的資源（圖片、SVG） | Step 4：需要資源時 |

#### mcp-fs 輔助工具

- 建立與修改組件檔案時使用

---

## 5) 禁止事項

| 禁止 | 替代方案 |
|------|----------|
| fetch() / axios | 改用 prop 傳入資料 |
| useStore() / Pinia action | 改用 emit 向上通知 |
| router.push() | 改用 emit 通知父層導頁 |
| v-if 加商業判斷 | 改用 computed prop |
| API response mapping | 由父層容器處理後傳入 |

---

## 6) 完成定義（DoD）

### ✅ 視覺保真檢查（必須通過）

- [ ] **版面精確**：Flex 方向、對齐、寬高比 — 與 Figma 截圖完全一致
- [ ] **間距精確**：Padding、Margin、Gap — 精確到 px
- [ ] **文字精確**：字號、字重、行高、letter-spacing — 完全複製
- [ ] **邊框 & 圓角精確**：寬度、半徑、顏色 — 精確到 px
- [ ] **色彩精確**：RGB / 16 進位 — 完全匹配 Figma（可透過設計令牌）
- [ ] **陰影精確**：offset、blur、spread、color — 完全複製
- [ ] **互動狀態一致**：Hover、Active、Disabled 的視覺變化可追溯
- [ ] **響應式正確**：所有斷點行為符合 Figma constraints
- [ ] **資源完整**：圖片、icon、SVG 正確顯示，無占位符

### ✅ 代碼品質檢查

- [ ] 所有互動點都有對應 props 或 emits
- [ ] 無 API 呼叫與業務邏輯混入
- [ ] SCSS 採 Mobile First 撰寫
- [ ] 巢狀不超過 4 層
- [ ] TypeScript 型別定義完整
- [ ] 已完成 node-to-component mapping
- [ ] 已完成 coverage self-check
- [ ] 所有核准範圍內的 node 均被標記為 implemented / partial / uncovered
- [ ] 所有資源失敗 fallback 已被記錄，沒有隱藏近似替代

### ✅ 交接清單

- [ ] 本地預覽視覺驗證通過
- [ ] 輸出 Handoff Payload 供下游 Agent（api-enrichment / logic-coder）使用
- [ ] Handoff Payload 含 `spec_path`、`approved_node_ids`、`covered_node_ids`、`uncovered_node_ids`、`asset_fallbacks`
- [ ] 組件檔案與 types.ts 已提交

---

## 7) 常見模式

詳見 references/component-patterns.md（Table、Form、Card、Modal 的切版範本）