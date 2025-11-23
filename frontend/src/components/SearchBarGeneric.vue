<template>
  <div class="search-bar-generic" ref="containerRef">
    <InputText
      ref="inputRef"
      type="text"
      :placeholder="placeholder"
      v-model="searchQuery"
      :size="size"
      @input="handleInput"
      @keyup.enter="handleEnter"
      @keydown.down.prevent="navigateDown"
      @keydown.up.prevent="navigateUp"
      @keydown.escape="closeDropdown"
      @blur="handleBlur"
      :class="[error ? 'p-invalid searchbar-input' : 'searchbar-input']"
    />

    <!-- Dropdown Results -->
    <div v-if="showDropdown && results.length > 0" class="search-dropdown">
      <div
        v-for="(item, index) in results"
        :key="index"
        class="search-item"
        :class="{ 'search-item-active': index === activeIndex }"
        @mousedown.prevent="selectItem(item)"
        @mouseenter="activeIndex = index"
      >
        <slot name="item" :item="item">
          {{ displayLabel(item) }}
        </slot>
      </div>
    </div>

    <!-- Error Message -->
    <small v-if="error" class="p-error">{{ error }}</small>
  </div>
</template>

<script setup lang="ts" generic="T">
import InputText from 'primevue/inputtext'
import { ref, nextTick, type Ref } from 'vue'

interface Props {
  placeholder?: string
  size?: 'small' | 'large'
  searchMethod: (query: string) => Promise<T[]>
  validateQuery?: (query: string) => boolean | string
  displayLabel?: (item: T) => string
  clearOnSelect?: boolean
  debounceMs?: number
  minChars?: number
  focusElementRef?: Ref<HTMLElement | null>
}

const props = withDefaults(defineProps<Props>(), {
  placeholder: 'Search...',
  size: 'small',
  clearOnSelect: false,
  debounceMs: 300,
  minChars: 1,
})

const emit = defineEmits<{
  itemSelected: [item: T]
  error: [error: string]
}>()

// Reactive state
const searchQuery = ref('')
const results = ref<T[]>([])
const showDropdown = ref(false)
const error = ref('')
const activeIndex = ref(-1)
const isLoading = ref(false)
const inputRef = ref<InstanceType<typeof InputText> | null>(null)
const containerRef = ref<HTMLDivElement | null>(null)

let debounceTimer: ReturnType<typeof setTimeout> | null = null

// Default display label function
const displayLabel = (item: T): string => {
  if (props.displayLabel) {
    return props.displayLabel(item)
  }
  // Default: try to access common properties
  if (typeof item === 'string') return item
  if (typeof item === 'object' && item !== null) {
    const obj = item as Record<string, unknown>
    const name = obj.name
    const label = obj.label
    const title = obj.title

    if (typeof name === 'string') return name
    if (typeof label === 'string') return label
    if (typeof title === 'string') return title

    return String(item)
  }
  return String(item)
}

// Validate query
const validateSearch = (query: string): boolean => {
  error.value = ''

  if (!query || query.length < props.minChars) {
    return false
  }

  if (props.validateQuery) {
    const validationResult = props.validateQuery(query)
    if (validationResult === false) {
      error.value = 'Invalid search query'
      emit('error', error.value)
      return false
    }
    if (typeof validationResult === 'string') {
      error.value = validationResult
      emit('error', error.value)
      return false
    }
  }

  return true
}

// Perform search
const performSearch = async (query: string) => {
  if (!validateSearch(query)) {
    results.value = []
    showDropdown.value = false
    return
  }

  isLoading.value = true
  error.value = ''

  try {
    const searchResults = await props.searchMethod(query)
    results.value = searchResults
    showDropdown.value = searchResults.length > 0
    activeIndex.value = -1
  } catch (err) {
    error.value = err instanceof Error ? err.message : 'Search failed'
    emit('error', error.value)
    results.value = []
    showDropdown.value = false
  } finally {
    isLoading.value = false
  }
}

const handleInput = () => {
  if (debounceTimer) {
    clearTimeout(debounceTimer)
  }

  debounceTimer = setTimeout(() => {
    performSearch(searchQuery.value)
  }, props.debounceMs)
}

// Handle enter key
const handleEnter = () => {
  if (activeIndex.value >= 0 && results.value[activeIndex.value]) {
    selectItem(results.value[activeIndex.value])
  }
}

const navigateDown = () => {
  if (!showDropdown.value) return
  activeIndex.value = Math.min(activeIndex.value + 1, results.value.length - 1)
}

const navigateUp = () => {
  if (!showDropdown.value) return
  activeIndex.value = Math.max(activeIndex.value - 1, -1)
}

const closeDropdown = () => {
  showDropdown.value = false
  activeIndex.value = -1
}

const handleBlur = () => {
  setTimeout(() => {
    closeDropdown()
  }, 200)
}

const selectItem = async (item: T) => {
  emit('itemSelected', item)

  if (props.clearOnSelect) {
    searchQuery.value = ''
  }

  closeDropdown()

  if (props.focusElementRef?.value) {
    await nextTick()
    props.focusElementRef.value.focus()
  }
}

defineExpose({
  focus: () => {
    if (inputRef.value) {
      const componentInstance = inputRef.value as { $el: HTMLInputElement }
      componentInstance.$el?.focus()
    }
  },
  clear: () => {
    searchQuery.value = ''
    results.value = []
    error.value = ''
    closeDropdown()
  },
})
</script>

<style scoped>
.search-bar-generic {
  position: relative;
  width: 100%;
  display: flex;
  flex-direction: column;
}

.search-dropdown {
  position: absolute;
  top: 100%;
  left: 0;
  right: 0;
  background-color: var(--color-background-primary);
  border: 1px solid var(--color-secondary);
  border-top: none;
  border-radius: 0 0 4px 4px;
  max-height: 300px;
  overflow-y: auto;
  z-index: 1000;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.search-item {
  padding: 10px 12px;
  cursor: pointer;
  transition: background-color 0.2s;
  background-color: var(--color-background-primary);
}

.search-item:hover,
.search-item-active {
  background-color: var(--color-background-primary);
  opacity: 0.9;
}

.searchbar-input {
  height: 36px;
}

.p-error {
  display: block;
  margin-top: 4px;
  color: #e24c4c;
  font-size: 0.875rem;
}
</style>
