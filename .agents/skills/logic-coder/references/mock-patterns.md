# Mock Patterns Reference

無 API 時的 Mock 建立規範與切換策略。

---

## 1) 目錄結構

```
src/services/
├── claimService.ts            # 真實 API 呼叫
└── __mock__/
    ├── claimService.mock.ts   # Mock 資料 + Mock service
    └── index.ts               # Mock 統一入口（視需要）
```

---

## 2) Mock Service 範本

```typescript
// src/services/__mock__/claimService.mock.ts
import type { ClaimRecord } from '@/types/claim'

// ─── Mock 資料 ───────────────────────────────────────
export const mockClaimRecords: ClaimRecord[] = [
  {
    id: 'CLM-001',
    claimTitle: '醫療理賠申請',
    claimStatus: 'pending',
    claimAmount: 15000,
    createdAt: '2024-01-15T08:00:00Z',
    updatedAt: '2024-01-15T08:00:00Z',
  },
  {
    id: 'CLM-002',
    claimTitle: '住院補助申請',
    claimStatus: 'approved',
    claimAmount: 30000,
    createdAt: '2024-01-10T08:00:00Z',
    updatedAt: '2024-01-12T10:00:00Z',
  },
  {
    id: 'CLM-003',
    claimTitle: '手術費用補助',
    claimStatus: 'rejected',
    claimAmount: 8500,
    createdAt: '2024-01-05T08:00:00Z',
    updatedAt: '2024-01-08T14:30:00Z',
  },
]

// ─── Mock Functions ────────────────────────────────
const MOCK_DELAY = 800 // ms

function delay(ms: number) {
  return new Promise(resolve => setTimeout(resolve, ms))
}

export async function fetchClaimRecords(): Promise<ClaimRecord[]> {
  await delay(MOCK_DELAY)
  return [...mockClaimRecords]
}

export async function fetchClaimById(id: string): Promise<ClaimRecord> {
  await delay(MOCK_DELAY)
  const record = mockClaimRecords.find(r => r.id === id)
  if (!record) throw new Error(`找不到理賠紀錄 ${id}`)
  return { ...record }
}
```

---

## 3) 環境切換策略（VITE_MOCK_MODE）

在 `.env.development` 加入：

```env
VITE_MOCK_MODE=true
VITE_API_BASE_URL=https://api.example.com
```

Service 層統一切換：

```typescript
// src/services/claimService.ts
import { fetchClaimRecords as fetchMock } from './__mock__/claimService.mock'

const USE_MOCK = import.meta.env.VITE_MOCK_MODE === 'true'

export async function fetchClaimRecords(): Promise<ClaimRecord[]> {
  if (USE_MOCK) return fetchMock()

  const res = await fetch(`${import.meta.env.VITE_API_BASE_URL}/claims`)
  if (!res.ok) throw new ApiError(res.status, '取得理賠紀錄失敗')
  return res.json()
}
```

---

## 4) 資料結構協商輸出格式

當缺少 API schema 時，使用此格式輸出提案給使用者確認：

```
⏸ 斷點：資料結構確認

根據展示元件的 props，我推導出以下 API Response 結構：

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[Type] ClaimRecord
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
欄位名稱         型別        對應 props    說明
──────────────────────────────────────────
id              string      -             主鍵
claimTitle      string      title         理賠標題
claimStatus     string      status        狀態（需確認 enum）
claimAmount     number      amount        金額（元）
createdAt       string      -             建立時間 ISO 8601

⚠️  需要確認：
1. claimStatus 的實際 enum 值？（目前假設與 UI 一致）
2. 是否有分頁？（page / pageSize / total）
3. 欄位命名是否符合後端慣例（camelCase / snake_case）？

確認後請輸入 OK，或直接修正欄位後回覆。
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```