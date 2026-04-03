# 無 OpenAPI 時的資料模型反推提案

当您暂时没有 OpenAPI 文檔时，api-enrichment Skill 会根据 Figma 设计稿中的 UI 欄位，反推候选的資料結構。
以下是一份完整的提案范例（基於「理賠紀錄列表」）。

---

## 背景

**狀況**：vue3-layout 已完成切版，设计稿显示清晰，但後端 API 文檔尚未就绪。

**目標**：反推出合理的資料模型，交由工程師確認，供 logic-coder 後續實裝使用。

---

## 推導過程

### Step 1：從 spec.md 提取 UI 欄位

基於 ai-pm 產出的 spec.md，Section 4 和 Section 5 中列出的所有推測欄位：

| UI 位置 | 推測欄位名 | 推測型別 | 備註 |
|---|---|---|---|
| 卡片標題 | claim_title | string | 顯示於卡片標題區 |
| 卡片狀態標籤 | claim_status | enum | StatusBadge 呈現，有多種顏色 |
| 卡片金額 | claim_amount | number | 顯示及需格式化 |
| 卡片日期 | created_at | string | 需格式化為人類可讀 |
| 紀錄識別 | id | string | 用於詳情頁導航、取消操作 |

### Step 2：根據 UI 互動推敲額外欄位

從 spec.md Section 3（互動行為說明）推敲可能需要的欄位：

| 互動 | 推敲欄位 | 說明 |
|---|---|---|
| 「點擊『查看詳情』導至詳情頁」 | id (已有) | 用於構造 `/claims/:id` 路由 |
| 「點擊『取消申請』（僅 pending 狀態可見）」 | status (已有) | 前端判斷按鈕是否顯示 |
| 「分頁控制」| 無額外欄位，但需 API 提供 page/pageSize/total | API 層級參數 |
| 「進入頁面自動加載」 | 無 | 由 composable 層負責 |

### Step 3：構建候選資料模型

基於上述分析，構建以下候選結構：

```typescript
// 單筆理賠紀錄
interface CandidateClaimRecord {
  id: string                              // 紀錄 ID（用於導航和操作）
  title: string                           // 理賠標題
  status: 'pending' | 'approved' | 'rejected'  // 狀態（決定顏色和可操作性）
  amount: number                          // 金額（需格式化）
  createdAt: string                       // 建立日期（需格式化）
  
  // 以下欄位為假設（可能需要 API 補充）
  updatedAt?: string                      // 更新日期（可選）
  description?: string                    // 詳細說明（若 Figma 未見，可先設為可選）
}

// 列表 API Response
interface CandidateClaimsListResponse {
  page: number                            // 當前頁
  pageSize: number                        // 每頁筆數
  total: number                           // 總筆數
  items: CandidateClaimRecord[]           // 當前頁的理賠紀錄
}
```

---

## 📋 資料模型確認清單

請根據以下表格逐一確認或修正：

### 欄位確認

| # | 欄位名 | 推測型別 | 推測用途 | ✅ 是否正確 | 💬 修正意見 |
|---|---|---|---|---|---|
| 1 | id | string (UUID 或 numeric) | 紀錄識別,用於導頁和刪除 | ? | ? |
| 2 | title | string (1-200 字元) | 卡片標題呈現 | ? | ? |
| 3 | status | enum (pending/approved/rejected) | 狀態標籤呈現及按鈕顯示邏輯 | ? | ? |
| 4 | amount | number (正整數) | 金額呈現，需加千分位格式化 | ? | ? |
| 5 | createdAt | ISO 8601 string | 日期呈現，需格式化為 YYYY/MM/DD | ? | ? |
| 6 | updatedAt | ISO 8601 string (可選) | 是否需要展示「最後更新」信息？ | ? | ? |

### Enum 值確認

**status** — 候選狀態值：

| 推測值 | UI 呈現 | 顏色 | ✅ 確認 | 💬 修正 |
|---|---|---|---|---|
| pending | 審核中 | 黃色 | ? | ? |
| approved | 已核准 | 綠色 | ? | ? |
| rejected | 已拒絕 | 紅色 | ? | ? |

> 若後端使用不同的 enum 值（如 'PENDING', 'APPROVED', 'REJECTED' 大寫，或 'pending_review' 蛇形命名等），請註明，logic-coder 會建立 Mapping。

### 欄位命名風格確認

| 推測風格 | 例 | ✅ 確認 | 💬 修正 |
|---|---|---|---|
| camelCase（前端常用） | `createdAt`, `claim_title` | ? | 請提供後端使用的命名風格 |
| snake_case（某些後端使用） | `created_at`, `claim_title` | ? | |
| PascalCase（某些 C# 後端使用） | `CreatedAt`, `ClaimTitle` | ? | |

### 單位與格式確認

| 欄位 | 推測單位 / 格式 | ✅ 確認 | 💬 修正 |
|---|---|---|---|
| amount | 新台幣元（TWD） | ? | 非人民幣或其他幣別嗎？ |
| createdAt | ISO 8601 (e.g., "2024-01-20T08:00:00Z") | ? | 或者是 Unix timestamp？或其他格式？ |

---

## 分頁與篩選確認

### 分頁參數

| 參數 | 推測型別 | 推測預設值 | 推測最大值 | ✅ 確認 | 💬 修正 |
|---|---|---|---|---|---|
| page (或 pageNum) | integer | 1 | — | ? | ? |
| pageSize (或 limit) | integer | 20 | 100 | ? | ? |
| total | integer (響應中) | — | — | ? | 來自 API 響應？ |

### 篩選功能（可選）

根據 spec.md，Figma 中是否有篩選 UI？

| 篩選方式 | 推測參數名 | 推測值 | 實裝需求 | 💬 備註 |
|---|---|---|---|---|
| 按狀態篩選 | status | pending / approved / rejected | ⚠️ 如需實裝中文 UI，需選項 | Figma 可見篩選控制項嗎？ |
| 按日期範圍篩選 | createdAt_from / createdAt_to | ISO 8601 格式 | ⚠️ Figma 可見日期選擇器嗎？ | |
| 按金額範圍篩選 | amount_min / amount_max | number | ⚠️ Figma 可見金額輸入框嗎？ | |
| 排序 | sortBy / order | status(status) / amount(amount) / createdAt(createdAt) / asc(asc) / desc(desc) | ⚠️ Figma 可見排序控制嗎？ | |

> 若 Figma 中未明確顯示，可暫時不實裝，後續 sprint 補上。

---

## 推薦的資料型別定義（TypeScript）

基於上述推測，以下是 logic-coder 可直接使用的型別定義（待您最終確認後）：

```typescript
// ============ 基礎資料型別 ============

/**
 * 單筆理賠紀錄（API Response 中的單筆資料）
 */
export interface ClaimRecord {
  id: string                                    // UUID 或數字 ID
  title: string                                 // 1-200 字元
  status: 'pending' | 'approved' | 'rejected'  // 狀態值
  amount: number                                // 正整數，單位 CNY
  createdAt: string                             // ISO 8601，e.g., "2024-01-20T08:00:00Z"
  updatedAt?: string                            // 可選，最後更新時間
}

/**
 * 列表 API 回傳（含分頁信息）
 */
export interface ClaimsListResponse {
  page: number               // 當前頁（1-based）
  pageSize: number           // 每頁筆數
  total: number              // 總筆數
  items: ClaimRecord[]       // 當前頁資料
}

// ============ 前端 ViewModel（展示層） ============

/**
 * UI 層使用的 ViewModel（已格式化）
 */
export interface ClaimCardViewModel {
  id: string
  title: string
  status: 'pending' | 'approved' | 'rejected'  // 原值
  displayStatus: string                         // 格式化值：「審核中」/ 「已核准」/ 「已拒絕」
  amount: string                                // 格式化，如 "15,000"
  date: string                                  // 格式化，如 "2024/01/20"
}

// ============ 分頁參數 ============

/**
 * 列表查詢參數
 */
export interface ClaimsListParams {
  page?: number         // 預設 1
  pageSize?: number     // 預設 20
  status?: string       // 可選篩選
}

// ============ 頁面 State ============

/**
 * 頁面管理的 State
 */
export interface ClaimListPageState {
  isLoading: boolean
  error: string | null
  records: ClaimRecord[]
  currentPage: number
  pageSize: number
  total: number
}
```

---

## ✅ 確認清單

請複製以下清單，完成後回覆給我：

### 欄位確認
- [ ] id 欄位名正確，型別為 string
- [ ] title / claim_title 確認欄位名（後端使用哪個？）
- [ ] status / claim_status 確認欄位名，enum 值正確
- [ ] amount / claim_amount 確認欄位名，單位為 CNY
- [ ] createdAt / created_at 確認欄位名，格式為 ISO 8601
- [ ] 無額外欄位遺漏

### 命名風格
- [ ] 確認後端使用的命名風格（camelCase / snake_case / PascalCase）

### 分頁
- [ ] 確認分頁參數名稱（page / pageNum）、預設值、最大值
- [ ] 確認 total 在 API 響應中如何提供

### 篩選與排序（可選）
- [ ] 確認是否需要狀態篩選
- [ ] 確認是否需要日期範圍篩選
- [ ] 確認是否需要排序功能

### 最終確認
- [ ] 上述所有項目已確認或修正
- [ ] 准許 logic-coder 基於此模型進行實裝

---

## 下一步

確認完成後，請回覆：

```
✅ 資料模型已確認。請根據以下修正進行實裝：

[列出任何修正項，例如：]
- status 欄位使用 'PENDING' / 'APPROVED' / 'REJECTED'（大寫）
- 需新增 updatedAt 欄位，型別同 createdAt
- pageSize 預設值為 30，而非 20
- 需支援按 status 篩選，參數名為 claimStatus

我將納入 enriched-spec.md 中，交由 logic-coder 實裝。
```

或若選擇等待真實 API 文檔：

```
API 文檔已就緒，連結為：https://api.example.com/swagger.json

請直接補充完整的 enriched-spec.md，我會基於 API 確認所有欄位。
```

---

## 範例：修正場景

假設您回覆如下：

```
✅ 修正如下：

1. status 應為大寫：'PENDING', 'APPROVED', 'REJECTED'
2. 欄位名使用 snake_case：claim_title, claim_status, claim_amount, created_at
3. 新增欄位 manager_id (string)，代表理賠經理
4. amount 有小數點，應為 number (可能 15000.50)
5. 分頁參數確認為 page (1-based), limit (預設 20，最大 100)
```

api-enrichment 將據此更新 enriched-spec.md：

```typescript
interface ClaimRecord {
  id: string
  claim_title: string                          // 欄位名改為 snake_case
  claim_status: 'PENDING' | 'APPROVED' | 'REJECTED'  // 改為大寫
  claim_amount: number                         // 支援小數
  created_at: string
  manager_id: string                           // 新增欄位
}

interface ClaimsListParams {
  page?: number    // 確認為 1-based
  limit?: number   // 參數改名，預設 20，最大 100
}
```

然後進入斷點 B-C，等待工程師 Approve。
