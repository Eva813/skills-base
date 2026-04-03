# Spec Template 填寫範例

以下是一份完整的 `draft-spec.md` 範例，以「理賠紀錄列表」功能為例。
ai-pm Agent 在產出時應以此為格式對照基準。

---

# Draft Spec：理賠紀錄列表（ClaimRecords）

> 產生時間：2024-01-20
> 狀態：Draft — 待工程師審閱
> Figma：https://www.figma.com/file/AbCdEf/insurance-portal?node-id=123:456
> 說明：本規格基於 Figma 設計稿推導。API 對應與資料結構將由後續 api-enrichment Skill 補充。

---

## 1. 功能範圍與流程摘要

**功能目標**：保戶可以在個人專區查看所有歷史理賠申請紀錄，包含狀態追蹤與金額資訊。

**涵蓋頁面**：
- `/claims` — 理賠紀錄列表頁

**明確排除**：
- 新增理賠申請（另有獨立流程）
- 理賠詳情頁（本次 spec 只處理列表）
- 管理後台的理賠審核操作

**主要流程**：進入頁面 → 顯示理賠紀錄清單與詳細資訊 → 支援各項互動（點擊詳情、取消申請等）

> 實際的 API 端點、Request/Response 格式與錯誤碼處理將由 api-enrichment Skill 補充。

---

## 2. 元件拆分建議  ← 工程師重點審閱

| 元件名稱 | Figma Node ID | 層級 | 可重用性 | 說明 |
|---|---|---|---|---|
| ClaimRecordsContainer | — | Container | 頁面專屬 | 負責 API 呼叫、loading/error 控制，傳 props 給列表 |
| ClaimRecordsList | 123:460 | Feature | 頁面專屬 | 接收 items 陣列，渲染卡片清單 |
| ClaimCard | 123:470 | Base | 跨頁可重用 | 單筆理賠紀錄展示，含狀態 badge 與金額 |
| StatusBadge | 123:480 | Base | 跨頁可重用 | 狀態標籤（pending/approved/rejected），可獨立抽出 |
| EmptyState | 123:490 | Base | 跨頁可重用 | 空資料提示元件，接收 message prop |

> 拆分原則：Container 負責資料與邏輯，Presentational 只接收 props。

---

## 3. 互動行為說明  ← 工程師重點審閱

| 互動點 | 觸發條件 | 預期行為 | 對應 Figma Frame |
|---|---|---|---|
| 進入頁面 | 路由 mounted | 自動呼叫 GET /claims，顯示 loading skeleton | 123:456 |
| 點擊「查看詳情」 | 使用者點擊 ClaimCard 的詳情按鈕 | 導向 `/claims/:id` 詳情頁 | 123:500 |
| 點擊「取消申請」 | 使用者點擊 ClaimCard 的取消按鈕（僅 pending 狀態可見）| [Open Question] 是否需確認 Modal？ | 123:510 |
| 載入失敗 | API 回傳 5xx 或網路異常 | 顯示錯誤提示 + 重試按鈕 | 123:520 |
| 無理賠紀錄 | Response data 為空陣列 | 顯示 EmptyState 元件 | 123:530 |

---

## 4. 欄位與資料型別定義  ← 工程師重點審閱

### 4.1 推測的列表欄位（待 API 確認）

基於 UI 設計稿推敲的可能欄位。實際欄位名稱、型別、enum 值由 api-enrichment Skill 後續確認。

| UI 位置 | 推測的欄位名稱 | 推測的型別 | 說明 | 待確認項目 |
|---|---|---|---|---|
| 卡片標題 | claim_title | string | 理賠申請標題 | [Assumption] 欄位名稱與型別 |
| 卡片狀態標籤 | claim_status | enum | 狀態顯示（審核中/已核准/已拒絕） | [Assumption] enum 值有哪些？camelCase 還是 snake_case？ |
| 卡片金額 | claim_amount | number | 申請金額 | [Assumption] 單位？是否需格式化？ |
| 卡片日期 | created_at | string | 申請日期 | [Assumption] 時間格式？ |
| 紀錄識別 | id | string | 用於詳情頁導航 | [Assumption] 欄位名稱 |

> 📝  後續流程：api-enrichment Skill 將補充完整的 API Response Schema、欄位驗證規則、錯誤碼定義。

---

## 5. 待定數據模型清單 ← api-enrichment 將完善此區段

以下是根據 Figma 設計稿推敲出需要補充的資料結構與 API 對應。此清單將在 api-enrichment Skill 階段完善。

### 5.1 主列表 API 推測

| 推測項目 | UI 呈現 | 待補充內容 |
|---|---|---|
| **列表查詢 API** | 進入頁面時自動加載理賠清單 | Endpoint / HTTP Method / Request params / Response schema |
| **分頁處理** | Figma 未明確顯示分頁 UI | 是否需要分頁？每頁幾筆？是否有 page/pageSize 參數？ |
| **狀態過濾** | StatusBadge 顯示多個狀態 | 狀態值為何？是否需前端過濾或 API 篩選？ |
| **錯誤提示** | Section 3 提及「載入失敗」 | HTTP 4xx / 5xx 的處理方案？是否有通用 error 物件格式？ |

### 5.2 操作 API 推測（可選）

| 推測項目 | UI 互動 | 待補充內容 |
|---|---|---|
| **取消申請** | 點擊「取消申請」按鈕 | 該操作對應的 API？哪些狀態允許取消？ |
| **查看詳情** | 點擊卡片導頁 | 詳情頁的 API 端點？（另行 spec，非本頁範圍） |

> 實際的 API 對應與資料驗證規則，請參考後續補充的 `enriched-spec.md`。

---

## 6. 開放問題與假設  ← 工程師與 api-enrichment 的協商點

### Assumptions（推測，由 Figma 設計稿推導）

- [Assumption] 列表包含至少 5 筆理賠紀錄展示（Figma 樣本數）
- [Assumption] StatusBadge 使用顏色區分狀態（黃/綠/紅 對應的狀態值需確認）
- [Assumption] 金額欄位需顯示千分位格式
- [Assumption] 日期欄位需格式化顯示（YYYY/MM/DD）

### Open Questions（需要人類決策）

- [Open Question] 列表是否需要分頁控制？若是，預設每頁幾筆？
- [Open Question] StatusBadge 的狀態值為何？（pending/approved/rejected？還是其他？）
- [Open Question] 「取消申請」操作是否需要二次確認 Modal？
- [Open Question] 是否需要支援依狀態篩選或排序功能？Figma 中未見 UI 線索。
- [Open Question] API 錯誤時的 UI 提示文案需確認嗎？

> 💡 **提示**：以上 Open Questions 將由工程師在斷點 A 審閱時補充決策，
> 後續 api-enrichment Skill 會根據決策結果補充 API 對應與驗證規則。