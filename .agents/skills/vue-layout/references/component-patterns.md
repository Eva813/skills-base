# Component Patterns Reference

常用 Vue 3 + SCSS 切版模式，供切版 Agent 快速參照。

---

## 1) Card 元件

適用：ClaimCard、PolicyCard、MemoCard

```vue
<template>
  <article class="base-card" :class="{ 'base-card--loading': isLoading }">
    <header class="base-card__header">
      <h3 class="base-card__title">{{ title }}</h3>
      <span class="base-card__badge" :data-status="status">{{ status }}</span>
    </header>
    <div class="base-card__body">
      <slot />
    </div>
    <footer class="base-card__footer">
      <button class="base-card__action" @click="emit('click:detail', id)">
        查看詳情
      </button>
    </footer>
  </article>
</template>

<script setup lang="ts">
interface Props {
  id: string
  title: string
  status: string
  isLoading?: boolean
}
const props = withDefaults(defineProps<Props>(), { isLoading: false })
const emit = defineEmits<{
  (e: 'click:detail', id: string): void
}>()
</script>

<style lang="scss" scoped>
.base-card {
  border-radius: 8px;
  border: 1px solid #e0e0e0;
  padding: 16px;

  &--loading { opacity: 0.6; pointer-events: none; }

  &__header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 12px;
  }

  &__title { font-size: 16px; font-weight: 600; margin: 0; }

  &__badge {
    font-size: 12px;
    padding: 2px 8px;
    border-radius: 4px;

    &[data-status="approved"] { background: #e8f5e9; color: #2e7d32; }
    &[data-status="pending"]  { background: #fff8e1; color: #f57f17; }
    &[data-status="rejected"] { background: #ffebee; color: #c62828; }
  }

  &__footer { margin-top: 16px; }

  &__action {
    padding: 8px 16px;
    border: 1px solid currentColor;
    border-radius: 4px;
    background: transparent;
    cursor: pointer;
  }
}
</style>
```

---

## 2) Table 元件

適用：ClaimRecords、PolicyRecords、MemoRecords

```vue
<template>
  <div class="data-table">
    <div class="data-table__wrapper">
      <table>
        <thead>
          <tr>
            <th v-for="col in columns" :key="col.key">{{ col.label }}</th>
            <th v-if="hasActions">操作</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="row in rows" :key="row.id">
            <td v-for="col in columns" :key="col.key">{{ row[col.key] }}</td>
            <td v-if="hasActions">
              <slot name="actions" :row="row" />
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup lang="ts">
interface Column { key: string; label: string }
interface Props {
  columns: Column[]
  rows: Record<string, unknown>[]
  hasActions?: boolean
}
withDefaults(defineProps<Props>(), { hasActions: false })
</script>

<style lang="scss" scoped>
.data-table {
  width: 100%;

  &__wrapper {
    overflow-x: auto;

    table {
      width: 100%;
      border-collapse: collapse;

      th, td {
        padding: 12px 16px;
        text-align: left;
        border-bottom: 1px solid #e0e0e0;
      }

      th { font-weight: 600; background: #f5f5f5; }
    }
  }
}
</style>
```

---

## 3) Form 元件

適用：表單類 UI，只暴露 modelValue 與 update:modelValue

```vue
<template>
  <div class="base-input">
    <label v-if="label" :for="inputId" class="base-input__label">
      {{ label }}
      <span v-if="required" class="base-input__required">*</span>
    </label>
    <input
      :id="inputId"
      class="base-input__field"
      :class="{ 'base-input__field--error': error }"
      :value="modelValue"
      :placeholder="placeholder"
      @input="emit('update:modelValue', ($event.target as HTMLInputElement).value)"
    />
    <p v-if="error" class="base-input__error">{{ error }}</p>
  </div>
</template>

<script setup lang="ts">
import { computed } from 'vue'
interface Props {
  modelValue: string
  label?: string
  placeholder?: string
  required?: boolean
  error?: string
  id?: string
}
const props = defineProps<Props>()
const emit = defineEmits<{ (e: 'update:modelValue', value: string): void }>()
const inputId = computed(() => props.id ?? `input-${Math.random().toString(36).slice(2)}`)
</script>

<style lang="scss" scoped>
.base-input {
  display: flex;
  flex-direction: column;
  gap: 4px;

  &__label { font-size: 14px; font-weight: 500; }
  &__required { color: #d32f2f; margin-left: 2px; }

  &__field {
    padding: 8px 12px;
    border: 1px solid #bdbdbd;
    border-radius: 4px;
    font-size: 14px;
    outline: none;
    transition: border-color 0.2s;

    &:focus { border-color: #1976d2; }
    &--error { border-color: #d32f2f; }
  }

  &__error { font-size: 12px; color: #d32f2f; margin: 0; }
}
</style>
```

---

## 4) Modal 元件

適用：對話框類，透過 modelValue 控制顯示

```vue
<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="modelValue" class="modal-overlay" @click.self="emit('update:modelValue', false)">
        <div class="modal" role="dialog" aria-modal="true">
          <header class="modal__header">
            <slot name="header" />
            <button class="modal__close" @click="emit('update:modelValue', false)">✕</button>
          </header>
          <div class="modal__body">
            <slot />
          </div>
          <footer v-if="$slots.footer" class="modal__footer">
            <slot name="footer" />
          </footer>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup lang="ts">
defineProps<{ modelValue: boolean }>()
const emit = defineEmits<{ (e: 'update:modelValue', value: boolean): void }>()
</script>

<style lang="scss" scoped>
.modal-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.modal {
  background: #fff;
  border-radius: 8px;
  width: min(90vw, 560px);
  max-height: 80vh;
  display: flex;
  flex-direction: column;

  &__header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 16px 24px;
    border-bottom: 1px solid #e0e0e0;
  }

  &__body { padding: 24px; overflow-y: auto; flex: 1; }

  &__footer {
    padding: 16px 24px;
    border-top: 1px solid #e0e0e0;
    display: flex;
    justify-content: flex-end;
    gap: 8px;
  }

  &__close {
    background: none;
    border: none;
    cursor: pointer;
    font-size: 18px;
    line-height: 1;
  }
}

.modal-enter-active, .modal-leave-active { transition: opacity 0.2s; }
.modal-enter-from, .modal-leave-to { opacity: 0; }
</style>
```