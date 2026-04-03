---
name: api-enrichment
description: >
  API 與數據規格補充 Skill。當完成 UI 切版預覽後（vue3-layout 斷點 B），需要補充 API、數據模型、State 結構時使用。
  根據 spec.md 中的「待定數據模型清單」+ OpenAPI 文檔，補充完整的 API 對應表、State 結構、Error 處理方案，
  產出 `enriched-spec.md` 供 logic-coder 實裝使用。
  若無 API 文檔，會主動產出「數據模型候選提案」讓使用者確認。
  觸發情境：「補充 API」、「確認資料格式」、「補完規格」、「有 Swagger 需要對齊」、「無 API 先補數據結構」。
---

# api-enrichment Skill — API 與數據規格補充

## 角色定位

你現在是 **API PM / 數據規格補充 Agent**，是 `ai-pm → vue3-layout → api-enrichment → logic-coder` workflow 的第三棒。  
你的任務：在 UI 切版完成後，補充 OpenAPI 對應、數據模型、State 結構、Error 處理方案，  
產生完整的 `enriched-spec.md`，讓 logic-coder 可以直接注入 API 邏輯。

---

## 執行 SOP

### Step 0：確認輸入來源

收集以下資訊，優先從 `vue3-layout` handoff payload 或用戶提供的信息獲取：

```
來自 Handoff Payload（斷點 B）：
- spec_path          → spec.md 路徑（來自 ai-pm + 工程師 Approve）
- component_paths    → 已切版的元件清單
- figma_node_ids     → 對應的 Figma Node IDs
- preview_verified   → 視覺預覽已確認

新增提供：
- api_documentation  → OpenAPI URL 或 Swagger JSON（可選）
- figma_link         → Figma 設計稿連結（同步設計稿版本）
```

若缺少 `spec.md`，輸出 blocked 狀態並停止執行。

---

### Step 1：讀取現有 spec.md

從 ai-pm 的產出讀取 spec.md，特別關注：

- **Section 4**：推測的欄位定義（UI 位置、推測欄位名、推測型別）
- **Section 5**：待定數據模型清單（列出需要 API 補充的項目）
- **Section 6**：Open Questions（列出待決策事項）

記錄所有 `[Assumption]` 和 `[Open Question]` 標記，這些是後續確認點。

---

### Step 2：確認 API 文檔可用性

#### 情況 A：用戶提供了 OpenAPI URL 或 Swagger JSON

執行 **Step 2a**（API 解析模式）。

#### 情況 B：用戶未提供 API 文檔

輸出提示訊息：

```
📋 未收到 API 文檔。

我可以：
1. 根據 spec.md 中的 UI 欄位反推「資料模型候選提案」，請你確認後繼續
2. 等待您提供 OpenAPI / Swagger 文檔，再補充完整規格

請選擇方案 1 或 2，或貼上 API 文檔連結。
```

- 若選方案 1，進入 **Step 2b**（無 API 提案模式）
- 若提供 API 文檔，進入 **Step 2a**
- 若無明確選擇，繼續等待

---

### Step 2a：解析 OpenAPI（有 API 時）

調用 `mcp-openapi` 或手動解析 Swagger JSON，依序提取：

1. **相關 Endpoint 清單** — 篩選與 spec.md 功能相關的 API
2. **Request Payload Schema** — 每個欄位的名稱、型別、是否必填、格式限制
3. **Response Schema** — 回傳資料結構與欄位型別（詳細定義）
4. **錯誤碼定義** — HTTP status code、error code、error message 格式
5. **API 間依賴關係** — 例如分頁參數、認證需求等

> ⚠️ 只記錄 OpenAPI 文件中**確實存在**的資訊，不得憑空假設。

---

### Step 2b：反推數據模型（無 API 時）

根據 spec.md Section 4 的「推測欄位」和 Section 5 的「待定項目」，反推候選資料結構：

#### 步驟 1：提取 UI 所需欄位

從 Section 4 的表格中，列出所有推測欄位：

```typescript
// 例：基於 ClaimCard 的 props 反推
interface CandidateClaimRecord {
  id: string                          // 推測：用於識別
  claim_title?: string                // 推測：卡片標題
  claim_status?: 'pending' | 'approved' | 'rejected'  // 推測：狀態值
  claim_amount?: number               // 推測：金額
  created_at?: string                 // 推測：日期時間
}
```

#### 步驟 2：輸出「資料模型候選提案」

以表格形式展示，讓用戶確認或修正：

| 推測欄位名 | 推測型別 | UI 對應 | 備註 |
|---|---|---|---|
| id | string | — | 紀錄識別，推測為 UUID 或 numeric ID |
| claim_title | string | 卡片標題 | 推測 1-200 字元 |
| claim_status | enum | 狀態標籤 | 推測 pending/approved/rejected，需確認實際值 |
| claim_amount | number | 金額顯示 | 推測為整數或小數，單位需確認 |
| created_at | ISO 8601 | 日期顯示 | 推測為 ISO 格式時間字串 |

#### 步驟 3：等待用戶確認

```
⏸ 斷點：資料模型確認

我根據設計稿推敲出上述資料結構。請確認：

1. 欄位名稱是否符合後端命名慣例？（camelCase vs snake_case）
2. enum 值是否正確？
3. 有沒有疏漏的欄位？
4. 是否有分頁、排序等額外參數？

確認後請輸入 OK，我會繼續生成完整的 enriched-spec.md。
```

若用戶提供修正，更新提案並繼續。

---

### Step 3：對齐 API 與設計稿

將 Figma 中的 UI 元件與 API 欄位進行對應：

| UI 位置 | 對應 API 欄位（來自 OpenAPI） | 型別 | 需要 Mapping | 備註 |
|---|---|---|---|---|
| 卡片標題 | claim_title | string | ❌ 無 | API 直接傳值 |
| 狀態標籤 | claim_status | enum | ✅ 有 | API: 'pending' → UI: 審核中；'approved' → 已核准；'rejected' → 已拒絕 |
| 金額 | claim_amount | number | ✅ 有 | API: 原始值 → UI: 加千分位格式化 + 幣別符號 |
| 日期 | created_at | ISO 8601 | ✅ 有 | API: '2024-01-20T08:00:00Z' → UI: '2024/01/20' |

記錄所有推測和不確定項為 `[Assumption]` 或 `[Open Question]`。

---

### Step 4：規劃 State 結構草案

基於 API 與設計稿的對應，設計 Vue 3 + TypeScript 的 State 結構：

#### 4.1 API 回傳的 State

```typescript
interface ClaimListApiResponse {
  data: ClaimRecord[]       // 當前頁資料
  total: number             // 總筆數
  page?: number             // 當前頁（若 API 回傳）
  pageSize?: number         // 每頁筆數（若 API 回傳）
}

interface ClaimRecord {
  id: string
  claim_title: string
  claim_status: 'pending' | 'approved' | 'rejected'
  claim_amount: number
  created_at: string
}
```

#### 4.2 頁面 State（loading / error / pagination）

```typescript
interface ClaimListPageState {
  isLoading: boolean        // API 呼叫中
  error: string | null      // 錯誤訊息（若有）
  records: ClaimRecord[]    // 當前頁資料
  currentPage: number       // 當前頁（預設 1）
  pageSize: number          // 每頁筆數（預設 20）
  total: number             // 總筆數
}
```

#### 4.3 衍生 State（computed）

```typescript
const isEmpty = computed(() => !isLoading.value && records.value.length === 0)
const totalPages = computed(() => Math.ceil(total.value / pageSize.value))
```

---

### Step 5：定義數據 Mapping 與轉換規則

從 API Response → UI Props 的具體對應邏輯：

```typescript
// API Response → ViewModel 轉換
interface ClaimCardViewModel {
  id: string
  title: string                    // 來自 claim_title
  status: 'pending' | 'approved' | 'rejected'  // 來自 claim_status，無需轉換
  displayStatus: string            // 轉換值：'審核中' / '已核准' / '已拒絕'
  amount: string                   // 來自 claim_amount，格式化後（加千分位）
  date: string                     // 來自 created_at，格式化為 YYYY/MM/DD
}

// Mapping 函式
function toViewModel(apiRecord: ClaimRecord): ClaimCardViewModel {
  const statusLabels = {
    'pending': '審核中',
    'approved': '已核准',
    'rejected': '已拒絕'
  }
  
  return {
    id: apiRecord.id,
    title: apiRecord.claim_title,
    status: apiRecord.claim_status,
    displayStatus: statusLabels[apiRecord.claim_status],
    amount: formatCurrency(apiRecord.claim_amount),  // 加千分位
    date: formatDate(apiRecord.created_at),          // YYYY/MM/DD
  }
}

// 輔助函式
const formatCurrency = (num: number): string => 
  new Intl.NumberFormat('zh-TW').format(num)

const formatDate = (isoStr: string): string => 
  new Date(isoStr).toLocaleDateString('zh-TW')
```

---

### Step 6：定義錯誤碼與異常處理方案

根據 OpenAPI 的錯誤碼定義，制定 UI 呈現策略：

| HTTP Status | API Error Code | 觸發條件 | UI 呈現 | 用戶可操作 |
|---|---|---|---|---|
| 401 | UNAUTHORIZED | 未登入或 token 過期 | 重新導向登入頁 | ❌ 無（自動跳轉） |
| 403 | FORBIDDEN | 無查看權限 | 「您沒有查看此資料的權限」靜態提示 | ❌ 無 |
| 400 | INVALID_PARAMS | 分頁參數不合法 | 回退到第一頁後重試 | ❌ 無（自動重試） |
| 404 | NOT_FOUND | 資源不存在 | 「紀錄不存在」提示 | ❌ 無 |
| 5xx | INTERNAL_ERROR | 伺服器異常 | 「系統暫時無法使用，請稍後再試」+ 重試按鈕 | ✅ 有（可點重試） |

#### 對應的 Composable 邏輯

```typescript
// 錯誤碼對應表
const errorMessages = {
  'UNAUTHORIZED': { code: 401, message: '請重新登入' },
  'FORBIDDEN': { code: 403, message: '您沒有查看此資料的權限' },
  'INVALID_PARAMS': { code: 400, message: '參數無效，已重置為第一頁' },
  'NOT_FOUND': { code: 404, message: '紀錄不存在' },
  'INTERNAL_ERROR': { code: 500, message: '系統暫時無法使用，請稍後再試' },
}

// 在 composable 中使用
const handleApiError = (status: number, errorCode?: string) => {
  if (status === 401) {
    // 重導至登入頁，由上層 router guard 處理
    window.location.href = '/login'
  } else if (status === 403) {
    error.value = errorMessages['FORBIDDEN'].message
  } else if (status === 400) {
    currentPage.value = 1
    reload()  // 自動重試第一頁
  } else if (status === 5xx) {
    error.value = errorMessages['INTERNAL_ERROR'].message
    // 留給使用者點重試按鈕
  }
}
```

---

### Step 7：檢查待決策項與假設

回顧 spec.md 中的所有 `[Open Question]` 和 `[Assumption]`，標記哪些已由 API 文檔回答：

| 原 Open Question | 決策方案 | 來源 |
|---|---|---|
| 列表是否需要分頁？ | 是，根據 API 支援 page/pageSize 參數 | OpenAPI 文檔 |
| 狀態值為何？ | pending/approved/rejected（已確認） | OpenAPI schema |
| 是否需要篩選功能？ | 否，Figma 未見相關 UI，暫不實裝 | 工程師決策 |
| API 錯誤時的提示文案？ | 見 Step 6 的錯誤碼對應表 | 本 Skill 補充 |

若仍有未決策項，輸出為待確認。

---

### Step 8：輸出 enriched-spec.md 與斷點 B-C

基於以上分析生成完整的 `enriched-spec.md`，結構如下：

```markdown
# Enriched Spec：{功能名稱}

> 產生時間：{日期}
> 狀態：Enriched — 基於 API 補充完整
> Figma：{URL}
> OpenAPI：{URL}

## 1. 功能範圍與流程摘要
（來自 spec.md，無變化）

## 2. 元件拆分建議
（來自 spec.md，無變化）

## 3. 互動行為說明
（來自 spec.md，無變化）

## 4. 欄位與資料型別定義
（已補充完整的 API 欄位定義）

## 5. API 對應表 ⭐️ 新增
（Endpoint、Request/Response schema、參數說明）

## 6. State 結構草案 ⭐️ 新增
（頁面 State、API State、衍生 State）

## 7. 數據 Mapping 與轉換規則 ⭐️ 新增
（API Response → UI Props 的具體邏輯）

## 8. Loading / Empty / Error 狀態
（每個狀態的 API 條件和 UI 呈現）

## 9. 錯誤碼與異常處理方案 ⭐️ 新增
（HTTP 狀態碼與 UI 對應表）

## 10. 開放問題與假設（已更新）
（標記已決策項，列出仍待確認項）
```

輸出後，顯示以下提示訊息進入斷點 B-C：

```
✅ enriched-spec.md 已產生。

📋 請審閱以下新增內容：
  - Section 5：API 對應表是否正確？
  - Section 6：State 結構是否合理？
  - Section 7：Mapping 邏輯是否清晰？
  - Section 9：Error 處理方案是否完整？

確認無誤後請回覆「Approve」，我將輸出交接 payload 給 logic-coder。
未收到 Approve 前，不會繼續任何下游產出。
```

---

## 品質守則

| 規則 | 說明 |
|---|---|
| 遵循 OpenAPI | 欄位名稱、型別、enum 值必須與 OpenAPI 文件一致 |
| 標記不確定 | 無法從 OpenAPI 推導的項目標記 `[Assumption]` |
| Mapping 清晰 | 所有 API → UI 的轉換邏輯必須明確寫出 |
| Error 完整 | 所有 API 可能的 HTTP status code 都需對應 UI 呈現 |
| State 實用 | State 結構應直接對應後續 logic-coder 的實裝需求 |
| 無 API 時主動提案 | 若缺 OpenAPI，不得停滯；應反推「資料模型候選提案」 |

---

## 完成定義（DoD）

- [ ] 收集到 spec.md、Figma 連結（必填）
- [ ] 若有 OpenAPI，成功解析並提取 endpoint、schema、錯誤碼
- [ ] 若無 OpenAPI，產出「資料模型候選提案」並等待用戶確認
- [ ] Section 5 API 對應表完整，所有欄位都有對應
- [ ] Section 6 State 結構清晰，包含分頁、loading、error state
- [ ] Section 7 Mapping 邏輯詳細，所有格式轉換都有函式示例
- [ ] Section 9 Error 碼表完整，涵蓋所有可能的 HTTP status
- [ ] enriched-spec.md 全部 10 個章節結構完整
- [ ] 已進入斷點 B-C 等待工程師 Approve

---

## 參考文件

- `references/enrichment-template.md` — 完整的 enriched-spec.md 填寫範例
- `references/data-structure-proposal.md` — 無 OpenAPI 時的數據模型反推範例
