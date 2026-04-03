# Error Handling Reference

Vue 3 前端錯誤處理規範，供 logic-coder Agent 參照。

---

## 1) ApiError Class

```typescript
// src/services/ApiError.ts
export class ApiError extends Error {
  constructor(
    public readonly status: number,
    message: string,
    public readonly code?: string,
  ) {
    super(message)
    this.name = 'ApiError'
  }

  get isClientError() { return this.status >= 400 && this.status < 500 }
  get isServerError() { return this.status >= 500 }
  get isUnauthorized() { return this.status === 401 }
  get isForbidden() { return this.status === 403 }
  get isNotFound() { return this.status === 404 }
}
```

---

## 2) 錯誤分類處理

```typescript
import { ApiError } from '@/services/ApiError'

function resolveErrorMessage(err: unknown): string {
  if (err instanceof ApiError) {
    if (err.isUnauthorized) return '請重新登入'
    if (err.isForbidden) return '您沒有執行此操作的權限'
    if (err.isNotFound) return '找不到相關資料'
    if (err.isClientError) return err.message || '請求資料有誤，請確認後再試'
    if (err.isServerError) return '系統暫時無法使用，請稍後再試'
  }
  if (err instanceof TypeError && err.message.includes('fetch')) {
    return '網路連線異常，請確認網路後重試'
  }
  console.error('[未知錯誤]', err)
  return '發生未預期的錯誤，請重新整理頁面'
}
```

---

## 3) Service 層 fetch 封裝

```typescript
// src/services/http.ts
import { ApiError } from './ApiError'

export async function http<T>(url: string, options?: RequestInit): Promise<T> {
  const controller = new AbortController()
  const timeoutId = setTimeout(() => controller.abort(), 10_000) // 10s 超時

  try {
    const res = await fetch(url, {
      ...options,
      signal: controller.signal,
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers,
      },
    })

    if (!res.ok) {
      const body = await res.json().catch(() => ({}))
      throw new ApiError(res.status, body.message ?? res.statusText, body.code)
    }

    return res.json() as Promise<T>
  } catch (err) {
    if ((err as Error).name === 'AbortError') {
      throw new ApiError(408, '請求逾時，請重試')
    }
    throw err
  } finally {
    clearTimeout(timeoutId)
  }
}
```

---

## 4) 重試策略

```typescript
// 指數退避重試（適用於 5xx / 網路異常）
export async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3,
  baseDelay = 1000,
): Promise<T> {
  let lastErr: unknown

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn()
    } catch (err) {
      lastErr = err
      // 只重試 5xx 和網路錯誤，不重試 4xx
      if (err instanceof ApiError && err.isClientError) throw err
      if (attempt < maxRetries - 1) {
        await new Promise(r => setTimeout(r, baseDelay * 2 ** attempt))
      }
    }
  }
  throw lastErr
}
```

---

## 5) 全域錯誤處理（main.ts）

```typescript
// src/main.ts
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.config.errorHandler = (err, instance, info) => {
  console.error('[Vue Global Error]', { err, info })
  // 可接入 Sentry 或其他監控服務
}

app.config.warnHandler = (msg, instance, trace) => {
  if (import.meta.env.DEV) {
    console.warn('[Vue Warning]', msg, trace)
  }
}
```