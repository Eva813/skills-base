# Enriched Spec Template 填寫範例

以下是一份完整的 `enriched-spec.md` 範例，基於「理賠紀錄列表」功能、結合 OpenAPI 文檔後的完整規格。

---

# Enriched Spec：理賠紀錄列表（ClaimRecords）

> 產生時間：2024-01-22
> 狀態：Enriched — 基於 API 補充完整
> Figma：https://www.figma.com/file/AbCdEf/insurance-portal?node-id=123:456
> OpenAPI：https://api.example.com/swagger/v1/swagger.json
> 說明：補充了 API 對應、State 結構、Error 處理。

---

## 1. 功能範圍與流程摘要

**功能目標**：保戶可以在個人專區查看所有歷史理賠申請紀錄，包含狀態追蹤與金額資訊。

**涵蓋頁面**：
- `/claims` — 理賠紀錄列表頁

**明確排除**：
- 新增理賠申請（另有獨立流程）
- 理賠詳情頁（本次 spec 只處理列表）
- 管理後台的理賠審核操作

**主要流程**：進入頁面 → API 取得列表 → 顯示卡片清單 → 支援分頁與互動

---

## 2. 元件拆分建議  ← 工程師重點審閱

| 元件名稱 | Figma Node ID | 層級 | 可重用性 | 說明 |
|---|---|---|---|---|
| ClaimRecordsContainer | — | Container | 頁面專屬 | 負責 API 呼叫、loading/error 控制，傳 props 給列表 |
| ClaimRecordsList | 123:460 | Feature | 頁面專屬 | 接收 items 陣列，渲染卡片清單 |
| ClaimCard | 123:470 | Base | 跨頁可重用 | 單筆理賠紀錄展示，含狀態 badge 與金額 |
| StatusBadge | 123:480 | Base | 跨頁可重用 | 狀態標籤（審核中/已核准/已拒絕），可獨立抽出 |
| EmptyState | 123:490 | Base | 跨頁可重用 | 空資料提示元件，接收 message prop |

> 拆分原則：Container 負責資料與邏輯，Presentational 只接收 props。

---

## 3. 互動行為說明  ← 工程師重點審閱

| 互動點 | 觸發條件 | 預期行為 | 對應 Figma Frame |
|---|---|---|---|
| 進入頁面 | 路由 mounted | 自動呼叫 GET /v1/claims，顯示 loading skeleton | 123:456 |
| 點擊「查看詳情」 | 使用者點擊 ClaimCard 的詳情按鈕 | 導向 `/claims/:id` 詳情頁 | 123:500 |
| 點擊「取消申請」 | 使用者點擊 ClaimCard 的取消按鈕（僅 pending 狀態可見） | 呼叫 DELETE /v1/claims/:id，刷新列表 | 123:510 |
| 點擊「下一頁」 | 使用者點擊分頁控制的「下一頁」按鈕 | 以 page + 1 呼叫 GET /v1/claims，刷新列表 | 123:535 |
| 載入失敗 | API 回傳 5xx 或網路異常 | 顯示錯誤提示 + 重試按鈕 | 123:520 |
| 無理賠紀錄 | Response data 為空陣列 | 顯示 EmptyState 元件 | 123:530 |

---

## 4. 欄位與資料型別定義  ← 工程師重點審閱

### 4.1 列表欄位（API Response → UI 對應）  ✅ 已完整確認

| 欄位名稱（API） | 顯示名稱（UI） | 型別 | 說明 | 來源 |
|---|---|---|---|---|
| id | — | string | 理賠紀錄主鍵 | OpenAPI: claim_record.id (UUID) |
| claim_title | 理賠標題 | string | 顯示於卡片標題 | OpenAPI: claim_record.title (1-200 chars) |
| claim_status | 狀態 | enum | 顯示於 StatusBadge | OpenAPI: claim_record.status (pending/approved/rejected) |
| claim_amount | 申請金額 | number | 單位：元，顯示時加千分位 | OpenAPI: claim_record.amount (integer, CNY) |
| created_at | 申請日期 | string (ISO 8601) | 格式化為 YYYY/MM/DD | OpenAPI: claim_record.createdAt (ISO 8601) |

### 4.2 請求參數（Request Query）

| 參數名稱 | 型別 | 必填 | 預設值 | 說明 |
|---|---|---|---|---|
| page | integer | ❌ | 1 | 當前頁數 |
| pageSize | integer | ❌ | 20 | 每頁筆數（API 支援最多 100） |
| status | string (enum) | ❌ | — | 篩選狀態（pending/approved/rejected），可選 |

> [Assumption] status 篩選參數由 API 提供，前端可選傳多個值

---

## 5. API 對應表  ⭐️ 新增

| 功能 | Method | Endpoint | Request Params | Response 關鍵欄位 | 錯誤碼處理 |
|---|---|---|---|---|---|
| 取得理賠列表 | GET | /v1/claims | page (int), pageSize (int), status (enum) | data: ClaimRecord[], page: int, pageSize: int, total: int | 401: 重新登入 / 403: 無權限 / 400: 參數無效 / 5xx: 通用錯誤 |
| 取消理賠申請 | DELETE | /v1/claims/:id | — | — | 400: 狀態不允許取消 / 404: 找不到紀錄 / 5xx: 通用錯誤 |

### 5.1 列表查詢 API 詳細定義

**Endpoint**: `GET /v1/claims`

**Request**:
```typescript
interface ClaimsListRequest {
  page?: number              // 預設 1
  pageSize?: number          // 預設 20，最大 100
  status?: 'pending' | 'approved' | 'rejected'  // 可選篩選
}
```

**Response** (200 OK):
```typescript
interface ClaimsListResponse {
  code: 'SUCCESS'
  data: {
    page: number              // 當前頁
    pageSize: number          // 每頁筆數
    total: number             // 總筆數
    items: ClaimRecord[]      // 當前頁資料
  }
}

interface ClaimRecord {
  id: string                 // UUID
  title: string              // 1-200 字元
  status: 'pending' | 'approved' | 'rejected'
  amount: number             // 正整數，單位元
  createdAt: string          // ISO 8601 格式，e.g., "2024-01-20T08:00:00Z"
  // 注意：API 欄位名為 createdAt（camelCase），前端對應 created_at
}
```

**Error Response** (4xx / 5xx):
```typescript
interface ErrorResponse {
  code: 'UNAUTHORIZED' | 'FORBIDDEN' | 'INVALID_PARAMS' | 'INTERNAL_ERROR'
  message: string
}
```

### 5.2 取消申請 API 詳細定義

**Endpoint**: `DELETE /v1/claims/:id`

**Request**: 無 query/body，:id 為申請紀錄 ID

**Response** (200 OK):
```typescript
interface CancelResponse {
  code: 'SUCCESS'
  message: 'Claim cancelled successfully'
}
```

**Error Response**:
- 400: claim_status 不是 'pending'，無法取消
- 404: 申請紀錄不存在
- 5xx: 系統異常

---

## 6. State 結構草案  ⭐️ 新增

### 6.1 API 回傳的 State（ClaimRecord）

```typescript
interface ClaimRecord {
  id: string
  title: string
  status: 'pending' | 'approved' | 'rejected'
  amount: number
  createdAt: string  // ISO 8601
}
```

### 6.2 頁面 State（ClaimListPageState）

```typescript
interface ClaimListPageState {
  isLoading: boolean              // API 呼叫中
  error: string | null            // 錯誤訊息
  records: ClaimRecord[]          // 當前頁的理賠紀錄
  currentPage: number             // 當前分頁（預設 1）
  pageSize: number                // 每頁筆數（預設 20）
  total: number                   // 總筆數（來自 API）
  selectedStatusFilter?: string   // 篩選狀態（可選）
}
```

### 6.3 衍生 State（computed）

```typescript
// 是否顯示空狀態
const isEmpty = computed(() => 
  !isLoading.value && records.value.length === 0
)

// 總頁數
const totalPages = computed(() => 
  Math.ceil(total.value / pageSize.value)
)

// 是否可點「下一頁」
const hasNextPage = computed(() => 
  currentPage.value < totalPages.value
)

// 是否可點「上一頁」
const hasPreviousPage = computed(() => 
  currentPage.value > 1
)
```

---

## 7. 數據 Mapping 與轉換規則  ⭐️ 新增

### 7.1 API Response → ViewModel 轉換

```typescript
// 從 API 獲得的原始資料
interface ClaimRecord {
  id: string
  title: string
  status: 'pending' | 'approved' | 'rejected'
  amount: number
  createdAt: string  // ISO 8601
}

// 展示層需要的 ViewModel
interface ClaimCardViewModel {
  id: string
  title: string
  status: 'pending' | 'approved' | 'rejected'
  displayStatus: string          // 轉換值：「審核中」/ 「已核准」/ 「已拒絕」
  amount: string                 // 格式化：加千分位，e.g., "15,000"
  date: string                   // 格式化：YYYY/MM/DD，e.g., "2024/01/20"
}
```

### 7.2 Mapping 函式

```typescript
// 單筆轉換
function toViewModel(apiRecord: ClaimRecord): ClaimCardViewModel {
  const statusLabels: Record<string, string> = {
    'pending': '審核中',
    'approved': '已核准',
    'rejected': '已拒絕'
  }
  
  return {
    id: apiRecord.id,
    title: apiRecord.title,
    status: apiRecord.status,
    displayStatus: statusLabels[apiRecord.status],
    amount: formatCurrency(apiRecord.amount),
    date: formatDate(apiRecord.createdAt),
  }
}

// 列表轉換
function toViewModels(apiRecords: ClaimRecord[]): ClaimCardViewModel[] {
  return apiRecords.map(toViewModel)
}

// 輔助函式：金額格式化（加千分位）
function formatCurrency(num: number): string {
  return new Intl.NumberFormat('zh-TW', {
    style: 'currency',
    currency: 'TWD',
    minimumFractionDigits: 0,
    maximumFractionDigits: 0
  }).format(num)
  // 例：15000 → "$15,000"
  // 若需移除貨幣符號，可用：
  // return num.toLocaleString('zh-TW')  // → "15,000"
}

// 輔助函式：日期格式化（ISO → YYYY/MM/DD）
function formatDate(isoStr: string): string {
  const date = new Date(isoStr)
  const year = date.getFullYear()
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const day = String(date.getDate()).padStart(2, '0')
  return `${year}/${month}/${day}`
  // 例："2024-01-20T08:00:00Z" → "2024/01/20"
}
```

### 7.3 Sorting / Filtering 邏輯（如需）

#### 按狀態篩選

```typescript
// 前端篩選（若 API 支援 status 參數）
const filterByStatus = (records: ClaimRecord[], status: string) => {
  if (!status) return records
  return records.filter(r => r.status === status)
}

// 或呼叫 API 篩選（推薦）
const fetchWithFilter = async (status?: string) => {
  const params = new URLSearchParams({
    page: String(currentPage.value),
    pageSize: String(pageSize.value),
  })
  if (status) params.append('status', status)
  
  const res = await fetch(`/v1/claims?${params}`)
  // ...
}
```

#### 按日期排序（若 API 支援）

> [Open Question] API 是否支援 sortBy / order 參數？目前 Figma 未見排序 UI。

---

## 8. Loading / Empty / Error 狀態

| 狀態 | 觸發條件 | UI 呈現 | 對應 Figma Node |
|---|---|---|---|
| Loading | API 呼叫中（isLoading: true） | Skeleton 卡片列表（3 筆） | 123:540 |
| Empty | Response data 為空陣列 + !isLoading | EmptyState：「目前沒有理賠紀錄」 | 123:530 |
| Error (401) | 未登入或 token 過期 | 重新導向登入頁（不顯示在列表頁） | — |
| Error (403) | 無查看權限 | 「您沒有查看此資料的權限」靜態提示 | 123:520 |
| Error (400) | 分頁參數不合法 | 自動回退至 page=1，重試 | — |
| Error (5xx) | 系統異常 | 「系統暫時無法使用，請稍後再試」+ 重試按鈕 | 123:520 |

---

## 9. 錯誤碼與異常處理方案  ⭐️ 新增

### 9.1 HTTP Status Code 對應表

| HTTP Status | API Error Code | 觸發條件 | UI 呈現 | 前端可操作 | 後續流程 |
|---|---|---|---|---|---|
| 200 | SUCCESS | API 正常回傳 | 正常顯示列表 | — | 無 |
| 400 | INVALID_PARAMS | 分頁參數超出範圍（如 page > totalPages） | 靜默回退至 page=1，重新呼叫 API | ❌ 無 | 自動重試 |
| 401 | UNAUTHORIZED | 未登入或 token 過期 | 重新導向登入頁 | ❌ 無 | 自動跳轉至 /login |
| 403 | FORBIDDEN | 用戶無查看此資料的權限 | 「您沒有查看此資料的權限」靜態文案 | ❌ 無 | 保持在列表頁，提示常駐 |
| 404 | NOT_FOUND | 資源不存在（罕見） | 「資料不存在」提示 | ❌ 無 | 提示常駐 |
| 5xx | INTERNAL_ERROR | 伺服器異常 | 「系統暫時無法使用，請稍後再試」+ 重試按鈕 | ✅ 有 | 用戶可點「重試」 |

### 9.2 Error Handling in Composable

```typescript
// 定義錯誤訊息對應表
const errorMessageMap = {
  '400': '參數無效，已重置',
  '401': '未登入，請重新登入',
  '403': '您沒有查看此資料的權限',
  '404': '資料不存在',
  '5xx': '系統暫時無法使用，請稍後再試'
}

// Composable 中的 error handler
async function load(pageNum: number = 1) {
  isLoading.value = true
  error.value = null
  try {
    const res = await fetch(`/v1/claims?page=${pageNum}&pageSize=${pageSize}`)
    
    if (!res.ok) {
      if (res.status === 401) {
        // 自動重導至登入
        window.location.href = '/login'
        return
      } else if (res.status === 403) {
        error.value = '您沒有查看此資料的權限'
      } else if (res.status === 400) {
        // 自動回退至第一頁
        currentPage.value = 1
        await load(1)
      } else if (res.status >= 500) {
        error.value = '系統暫時無法使用，請稍後再試'
      }
      return
    }
    
    const data = await res.json()
    records.value = data.items.map(toViewModel)
    total.value = data.total
    currentPage.value = pageNum
    
  } catch (err) {
    console.error('[useClaimRecords] 網路錯誤', err)
    error.value = '網路連線失敗，請檢查後重試'
  } finally {
    isLoading.value = false
  }
}
```

---

## 10. 開放問題與假設（已更新）  ← 工程師與 logic-coder 的協商點

### Assumptions（已確認，來自 OpenAPI 與 Figma）

- ✅ 列表需要分頁，預設每頁 20 筆（API 支援 pageSize 參數）
- ✅ 狀態值為 pending/approved/rejected（已確認）
- ✅ 金額單位為新台幣元，無需幣別轉換
- ✅ 日期欄位為 ISO 8601 格式，需格式化為 YYYY/MM/DD
- ✅ API 直接提供分頁資訊（page、pageSize、total）

### Open Questions（仍待決策）

- [Open Question] 「取消申請」操作是否需要二次確認 Modal？還是直接呼叫 API？（Figma 未給明確設計）
- [Open Question] 是否需要支援依狀態篩選或排序功能？（Figma 中未見 UI 線索，但 API 支援 status 參數）
- [Open Question] 分頁控制是否需要「跳頁輸入框」，還是只需上一頁/下一頁？（Figma 設計未明確）

> 💡 **提示**：以上 Open Questions 應由工程師在 code review 時決策，或由 logic-coder 在實裝過程中與設計確認。
