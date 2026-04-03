# Composable Patterns Reference

Vue 3 composable 的完整範本，供 logic-coder Agent 快速參照。

---

## 1) 基礎資料讀取（useXxxList）

```typescript
// src/composables/useClaimRecords.ts
import { ref, computed, onMounted } from 'vue'
import { fetchClaimRecords, ApiError } from '@/services/claimService'
import type { ClaimRecord, ClaimCardViewModel } from '@/types/claim'

function toViewModel(record: ClaimRecord): ClaimCardViewModel {
  return {
    id: record.id,
    title: record.claimTitle,
    status: record.claimStatus,
    amount: record.claimAmount,
  }
}

export function useClaimRecords() {
  const records = ref<ClaimRecord[]>([])
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  const viewModels = computed(() => records.value.map(toViewModel))
  const isEmpty = computed(() => !isLoading.value && records.value.length === 0)

  async function load() {
    isLoading.value = true
    error.value = null
    try {
      records.value = await fetchClaimRecords()
    } catch (err) {
      handleError(err, error)
    } finally {
      isLoading.value = false
    }
  }

  onMounted(load)

  return { viewModels, isLoading, error, isEmpty, reload: load }
}

// ─── 共用錯誤處理 ───────────────────────────────────
function handleError(err: unknown, error: Ref<string | null>) {
  if (err instanceof ApiError) {
    error.value = err.status === 403
      ? '您沒有查看此資料的權限'
      : err.message
  } else {
    console.error('[composable] 系統錯誤', err)
    error.value = '系統暫時無法使用，請稍後再試'
  }
}
```

---

## 2) 帶分頁的資料讀取（usePaginatedXxx）

```typescript
import { ref, computed, watch } from 'vue'

export function usePaginatedClaimRecords(pageSize = 20) {
  const records = ref<ClaimRecord[]>([])
  const currentPage = ref(1)
  const total = ref(0)
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  const totalPages = computed(() => Math.ceil(total.value / pageSize))
  const hasNextPage = computed(() => currentPage.value < totalPages.value)

  async function loadPage(page: number) {
    isLoading.value = true
    error.value = null
    try {
      const result = await fetchClaimRecords({ page, pageSize })
      records.value = result.data
      total.value = result.total
      currentPage.value = page
    } catch (err) {
      handleError(err, error)
    } finally {
      isLoading.value = false
    }
  }

  watch(currentPage, (page) => loadPage(page), { immediate: true })

  return {
    records,
    currentPage,
    total,
    totalPages,
    hasNextPage,
    isLoading,
    error,
    goToPage: (page: number) => { currentPage.value = page },
    nextPage: () => { if (hasNextPage.value) currentPage.value++ },
    prevPage: () => { if (currentPage.value > 1) currentPage.value-- },
  }
}
```

---

## 3) 單筆資料讀取 + 操作（useXxxDetail）

```typescript
import { ref } from 'vue'
import { fetchClaimById, cancelClaim, ApiError } from '@/services/claimService'

export function useClaimDetail(id: string) {
  const record = ref<ClaimRecord | null>(null)
  const isLoading = ref(false)
  const isCancelling = ref(false)
  const error = ref<string | null>(null)

  async function load() {
    isLoading.value = true
    error.value = null
    try {
      record.value = await fetchClaimById(id)
    } catch (err) {
      handleError(err, error)
    } finally {
      isLoading.value = false
    }
  }

  async function cancel() {
    if (!record.value) return
    isCancelling.value = true
    try {
      await cancelClaim(id)
      record.value = { ...record.value, claimStatus: 'cancelled' }
    } catch (err) {
      handleError(err, error)
    } finally {
      isCancelling.value = false
    }
  }

  load()

  return { record, isLoading, isCancelling, error, reload: load, cancel }
}
```

---

## 4) 表單提交 Composable（useXxxForm）

```typescript
import { reactive, ref } from 'vue'
import { submitClaim, ApiError } from '@/services/claimService'

interface ClaimFormData {
  title: string
  amount: number | null
  description: string
}

export function useClaimForm(onSuccess: () => void) {
  const form = reactive<ClaimFormData>({
    title: '',
    amount: null,
    description: '',
  })

  const errors = reactive<Partial<Record<keyof ClaimFormData, string>>>({})
  const isSubmitting = ref(false)
  const submitError = ref<string | null>(null)

  function validate(): boolean {
    Object.keys(errors).forEach(k => delete errors[k as keyof typeof errors])

    if (!form.title.trim()) errors.title = '請輸入理賠標題'
    if (!form.amount || form.amount <= 0) errors.amount = '請輸入有效金額'

    return Object.keys(errors).length === 0
  }

  async function submit() {
    if (!validate()) return

    isSubmitting.value = true
    submitError.value = null
    try {
      await submitClaim(form)
      onSuccess()
    } catch (err) {
      if (err instanceof ApiError && err.status === 422) {
        submitError.value = '資料格式錯誤，請確認後再試'
      } else {
        handleError(err, submitError)
      }
    } finally {
      isSubmitting.value = false
    }
  }

  return { form, errors, isSubmitting, submitError, submit }
}
```