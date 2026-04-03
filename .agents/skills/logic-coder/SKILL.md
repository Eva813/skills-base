---
name: logic-coder
description: "Vue 3 前端邏輯技能（邏輯 Agent）。接收 enriched-spec.md（含 API 對應表、State 結構定義、錯誤處理方案），將切版完成的純展示元件（Dumb Components）與 API / 狀態管理邏輯整合，產出可執行的容器層、services、composables。觸發情境包含：接續 api-enrichment 完成後的邏輯實作、「幫我串 API」、「實作資料邏輯」、「加上 loading / error 狀態」、「幫我寫 composable」、「實作 Pinia store」、提到 enriched-spec.md 或 component_paths handoff payload 的情境。"
---

# logic-coder Skill（前端邏輯 Agent）

## 1) 目標

將切版完成的無邏輯 UI 元件（Dumb Components）升級為可執行流程的前端功能：
- 狀態管理（Pinia / composable）
- API 呼叫 + 資料 mapping
- loading / error / empty 狀態控制
- 無 API 時的 mock 資料結構協商

---

## 2) 角色定義

- **你是**：前端邏輯 Agent（Vue 3 + TypeScript）
- **你的職責**：容器層、頁面層、composable、service、Pinia store
- **你不改**：展示元件（Dumb Components）的 template / SCSS
- **下游**：產出 `modified_files` + `diff_summary` 供 code review 使用

---

## 3) 執行流程

### Step 0：確認輸入來源

收集以下資訊，優先從 `api-enrichment` handoff payload 取得：

```
來自 Handoff Payload（斷點 B-C）：
- component_paths    → 要注入邏輯的元件清單（來自 vue3-layout）
- figma_node_ids     → 對照設計稿（不修改視覺）
- enriched_spec_path → enriched-spec.md 版本（已含 API 對應、State 結構、Error 處理）

輸入來源順序：
1. 優先：api-enrichment 完成後的 enriched-spec.md（含完整 API 定義）
2. 其次：若無 api-enrichment，使用 spec.md（來自 ai-pm + 工程師 Approve）
   - 此種情況下，無 API 資訊，將進入 Step 1a（mock 資料協商）
```

若缺少 component_paths 或 spec 文件，輸出 blocked 狀態並停止執行。

---

### Step 1：讀取 enriched-spec.md 中的數據結構定義

從 handoff payload 的 enriched-spec.md（斷點 B-C 已確認的規格）中提取以下資訊：

#### 1.1 讀取 API 對應表（enriched-spec.md 第 5 章）

若規格包含 API 資訊，提取：
- **Endpoint 清單** → URL、HTTP method、parameters
- **Request / Response Schema** → 欄位名稱、型別、預設值
- **錯誤碼對應表** → HTTP status → UI 提示訊息

```typescript
// 來自 enriched-spec.md 第 5 章「API 對應表」
// Example：
export interface ClaimRecord {
  id: string                              // 對應 props.id
  claimTitle: string                     // 對應 props.title
  claimStatus: 'pending' | 'approved' | 'rejected'  // 對應 props.status
  claimAmount: number                    // 對應 props.amount
  createdAt: ISO8601DateTime
}
```

#### 1.2 讀取 State 結構設計（enriched-spec.md 第 6 章）

從規格中提取 State type 定義與計算邏輯：

```typescript
// 來自 enriched-spec.md 第 6 章「State 結構草案」
export interface PageState {
  records: ClaimRecord[]
  currentPage: number
  pageSize: number
  totalCount: number
}

export interface DerivedState {
  isLoading: boolean
  hasError: boolean
  isEmpty: boolean
  errorMessage: string | null
}
```

#### 1.3 讀取數據 Mapping 規則（enriched-spec.md 第 7 章）

提取從 API Response → ViewModel 的轉換函式：

```typescript
// 來自 enriched-spec.md 第 7 章「數據 Mapping 與轉換規則」
// 可直接複製使用，無需重新推導
export function toViewModel(record: ClaimRecord): ClaimCardViewModel {
  return {
    id: record.id,
    title: record.claimTitle,
    status: mapClaimStatus(record.claimStatus),
    amount: formatCurrency(record.claimAmount, 'CNY'),
  }
}
```

#### 1.4 讀取錯誤處理方案（enriched-spec.md 第 8 章）

提取 HTTP status / 錯誤碼 → UI state 的對應關係：

```typescript
// 來自 enriched-spec.md 第 8 章「Loading / Empty / Error 狀態」
// status code 403 → "您沒有查看此資料的權限"
// status code 500 → "系統暫時無法使用，請稍後再試"
```

---

### Step 1b（若無 API）：讀取反向工程的數據模型提案

若 enriched-spec.md 由 data-structure-proposal 流程產出（無 OpenAPI 情況），則：
- 數據模型已由 api-enrichment 與工程師確認過
- 無需再次協商，直接使用 enriched-spec.md 中的 TypeScript type 定義
- 已內含 mock 資料結構，可直接匯入

---
  await new Promise(resolve => setTimeout(resolve, 800)) // 模擬網路延遲
  return mockClaimRecords

```

---

---

### Step 2：規劃實作架構

從 enriched-spec.md 確認資料結構與 API 對應後，輸出實作計畫：

```
實作計畫：

1. types/         → 新增 ClaimRecord、ApiResponse 型別（從 enriched-spec.md 複製）
2. services/      → claimService.ts（封裝 API 呼叫，基於 enriched-spec.md 第 5 章）
3. composables/   → useClaimRecords.ts（管理 loading/error/data，基於 enriched-spec.md 第 6 章狀態定義）
4. stores/        → claimStore.ts（若需跨頁共用狀態，推薦用 Pinia）
5. 容器層         → ClaimRecordsContainer.vue（注入邏輯，傳 props 給展示元件）

準備好開始實作？(Y/n)
```

---

### Step 3：實作順序

按以下順序實作，避免型別錯誤擴散：

#### 3.1 型別定義（types/）

```typescript
// src/types/claim.ts
export interface ClaimRecord {
  id: string
  claimTitle: string
  claimStatus: 'pending' | 'approved' | 'rejected'
  claimAmount: number
  createdAt: string
}

export interface PaginatedResponse<T> {
  data: T[]
  total: number
  page: number
  pageSize: number
}

// UI 用的 ViewModel（對應 props）
export interface ClaimCardViewModel {
  id: string
  title: string
  status: 'pending' | 'approved' | 'rejected'
  amount: number
}
```

#### 3.2 Service 層（services/）

```typescript
// src/services/claimService.ts
import type { ClaimRecord } from '@/types/claim'

const BASE_URL = import.meta.env.VITE_API_BASE_URL

export async function fetchClaimRecords(params?: {
  page?: number
  pageSize?: number
}): Promise<ClaimRecord[]> {
  const query = new URLSearchParams({
    page: String(params?.page ?? 1),
    pageSize: String(params?.pageSize ?? 20),
  })
  const res = await fetch(`${BASE_URL}/claims?${query}`)
  if (!res.ok) {
    const err = await res.json().catch(() => ({}))
    throw new ApiError(res.status, err.message ?? '請求失敗')
  }
  return res.json()
}

export class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message)
    this.name = 'ApiError'
  }
}
```

#### 3.3 Composable（composables/）

```typescript
// src/composables/useClaimRecords.ts
import { ref, onMounted } from 'vue'
import { fetchClaimRecords, ApiError } from '@/services/claimService'
import type { ClaimRecord } from '@/types/claim'

export function useClaimRecords() {
  const records = ref<ClaimRecord[]>([])
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  async function load(params?: { page?: number }) {
    isLoading.value = true
    error.value = null
    try {
      records.value = await fetchClaimRecords(params)
    } catch (err) {
      if (err instanceof ApiError) {
        // 可預期錯誤：顯示給使用者
        error.value = err.status === 403
          ? '您沒有查看此資料的權限'
          : err.message
      } else {
        // 系統錯誤：記錄並顯示通用訊息
        console.error('[useClaimRecords] 系統錯誤', err)
        error.value = '系統暫時無法使用，請稍後再試'
      }
    } finally {
      isLoading.value = false
    }
  }

  onMounted(load)

  return { records, isLoading, error, reload: load }
}
```

#### 3.4 資料 Mapping（ViewModel 轉換）

```typescript
// src/composables/useClaimRecords.ts（加入 mapping）
import type { ClaimRecord, ClaimCardViewModel } from '@/types/claim'

function toViewModel(record: ClaimRecord): ClaimCardViewModel {
  return {
    id: record.id,
    title: record.claimTitle,
    status: record.claimStatus,   // 若 enum 不同需轉換
    amount: record.claimAmount,
  }
}

// 在 composable 中 expose viewModels
const viewModels = computed(() => records.value.map(toViewModel))
```

#### 3.5 容器元件（Container）

```vue
<!-- src/components/ClaimRecords/ClaimRecordsContainer.vue -->
<template>
  <!-- loading 狀態 -->
  <div v-if="isLoading" class="state-loading">載入中...</div>

  <!-- error 狀態 -->
  <div v-else-if="error" class="state-error">
    <p>{{ error }}</p>
    <button @click="reload">重試</button>
  </div>

  <!-- empty 狀態 -->
  <div v-else-if="viewModels.length === 0" class="state-empty">
    目前沒有理賠紀錄
  </div>

  <!-- 正常資料 -->
  <ClaimRecordsList
    v-else
    :items="viewModels"
    @click:detail="handleDetail"
    @click:cancel="handleCancel"
  />
</template>

<script setup lang="ts">
import { computed } from 'vue'
import ClaimRecordsList from './ClaimRecordsList.vue'
import { useClaimRecords } from '@/composables/useClaimRecords'
import { useRouter } from 'vue-router'

const router = useRouter()
const { records, isLoading, error, reload } = useClaimRecords()

const viewModels = computed(() => records.value.map(r => ({
  id: r.id,
  title: r.claimTitle,
  status: r.claimStatus,
  amount: r.claimAmount,
})))

function handleDetail(id: string) {
  router.push(`/claims/${id}`)
}

function handleCancel(id: string) {
  // 呼叫 cancelClaim service（若 spec 有定義）
}
</script>
```

---

### Step 4：驗證清單（DoD Check）

完成實作後，自動驗證以下項目：

```
✅ DoD 驗證：

[ ] 主要流程可正常觸發 API 並更新畫面
[ ] loading / error / empty 三狀態皆有對應 UI
[ ] 展示元件的 template / SCSS 未被修改
[ ] ApiError 有區分 4xx（使用者可讀）與 5xx（通用提示）
[ ] 無 API 時有 mock service，且結構已與使用者確認
[ ] ViewModel mapping 函式有型別保護
[ ] 容器元件未混入 HTML / SCSS 業務設計
```

---

### Step 5：輸出 Handoff Payload

```json
{
  "modified_files": [
    "src/types/claim.ts",
    "src/services/claimService.ts",
    "src/services/__mock__/claimService.mock.ts",
    "src/composables/useClaimRecords.ts",
    "src/components/ClaimRecords/ClaimRecordsContainer.vue"
  ],
  "diff_summary": {
    "new_files": ["types/claim.ts", "services/claimService.ts", "composables/useClaimRecords.ts"],
    "modified_files": ["ClaimRecordsContainer.vue"],
    "untouched_files": ["ClaimRecordsList.vue（展示元件，未動）"]
  },
  "mock_mode": true,
  "api_endpoints": [],
  "spec_version": "enriched-spec.md v1.0"
}
```

---

## 4) MCP 工具使用指南

### mcp-openapi

```
# 取得 API schema
get_schema(swaggerUrl)

# 取得特定路徑的 schema
get_path_schema(swaggerUrl, '/claims', 'GET')

# 取得錯誤模型
get_error_models(swaggerUrl)
```

### mcp-fs

建立 / 修改以下類型的檔案：
- `src/types/*.ts`
- `src/services/*.ts`
- `src/composables/*.ts`
- `src/stores/*.ts`
- `src/components/**/*Container.vue`

### mcp-terminal（選用）

```bash
# TypeScript type check
npx vue-tsc --noEmit

# Build 驗證
npm run build
```

---

## 5) 錯誤處理規範

| 錯誤類型 | 處理方式 | 禁止事項 |
|---------|----------|---------|
| 4xx（客戶端錯誤）| 顯示可讀訊息給使用者 | 不可靜默忽略 |
| 5xx / 網路異常 | 顯示通用提示 + console.error | 不可只 log 不提示 |
| 驗證錯誤 | 對應欄位顯示錯誤 | 不可整頁 crash |
| 超時 | 顯示重試按鈕 | 不可無限等待 |

---

## 6) 禁止事項

| 禁止 | 原因 |
|------|------|
| 修改展示元件 template / SCSS | 保持 Dumb Component 乾淨 |
| 靜默吞錯（catch 空白） | 不可追蹤的 bug |
| 直接在 template 寫 API 呼叫 | 應抽入 composable |
| 未確認直接假設資料結構 | 無 API 時必須走協商流程 |
| 在展示元件 import store | 破壞元件可重用性 |

---

## 7) 參考文件

- `references/composable-patterns.md` — useXxx 的完整範本與測試寫法
- `references/error-handling.md` — ApiError class、重試策略、全域錯誤處理
- `references/mock-patterns.md` — Mock service 的建立規範與切換策略（VITE_MOCK_MODE）