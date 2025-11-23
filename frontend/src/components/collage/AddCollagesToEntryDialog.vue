<template>
  <div id="add-entries-to-collage-dialog">
    <div class="entries">
      <div v-for="(_entry, index) in newCollageEntries" :key="index" class="entry">
        <div class="inputs-container">
          <div class="collage-search-wrapper">
            <SearchBarGeneric
              v-if="!selectedCollages[index]"
              :placeholder="'Search collage by name or insert url'"
              :searchMethod="searchCollagesMethod"
              :displayLabel="(item) => item.name"
              :minChars="2"
              :debounceMs="300"
              @itemSelected="(collage) => onCollageSelected(collage, index)"
              @input="currentSearchIndex = index"
              :focusElementRef="noteInputRefs[index]"
            >
              <template #item="{ item }">
                <div class="collage-item">
                  <strong>{{ item.name }}</strong>
                  <span class="collage-id">#{{ item.id }}</span>
                </div>
              </template>
            </SearchBarGeneric>
            <div v-else class="selected-collage">
              <span class="collage-name">{{ selectedCollages[index].name }}</span>
              <Button icon="pi pi-times" size="small" text rounded @click="clearCollageSelection(index)" class="clear-btn" />
            </div>
          </div>

          <InputText :ref="(el) => setNoteRef(el, index)" class="note" :placeholder="t('collage.note')" v-model="newCollageEntries[index].note" />
        </div>

        <div class="buttons-container">
          <Button v-if="index === 0" @click="addCollageEntry" icon="pi pi-plus" size="small" />
          <Button v-if="newCollageEntries.length > 1" @click="removeCollageEntry(index)" icon="pi pi-minus" size="small" />
        </div>
      </div>
    </div>
    <div class="wrapper-center" style="margin-top: 10px">
      <Button :label="t('collage.add_collage_to_entry', 2)" size="small" :loading="loading" @click="sendCollageEntries" />
    </div>
  </div>
</template>

<script setup lang="ts">
import { InputText, Button } from 'primevue'
import { useI18n } from 'vue-i18n'
import SearchBarGeneric from '@/components/SearchBarGeneric.vue'
import {
  createCollageEntries,
  searchCollages,
  getCollage,
  type CollageEntry,
  type UserCreatedCollageEntry,
  type CollageSearchResult,
  type SearchCollagesQuery,
} from '@/services/api/collageService'
import { ref, onMounted, type Ref } from 'vue'

const { t } = useI18n()

const emit = defineEmits<{
  addedEntries: [CollageEntry[]]
}>()

const props = defineProps<{
  entryId: number
  entryType: 'TitleGroup' | 'Artist' | 'Entity' | 'MasterGroup'
}>()

const loading = ref(false)
const newCollageEntries = ref<UserCreatedCollageEntry[]>([])
const selectedCollages = ref<(CollageSearchResult | null)[]>([])
const noteInputRefs = ref<Ref<HTMLElement | null>[]>([])

const setNoteRef = (el: InstanceType<typeof InputText> | HTMLInputElement | null, index: number) => {
  if (el) {
    const inputElement = el.$el || el
    if (!noteInputRefs.value[index]) {
      noteInputRefs.value[index] = ref(inputElement)
    } else {
      noteInputRefs.value[index].value = inputElement
    }
  }
}

const extractCollageIdFromUrl = (input: string): number | null => {
  const urlPattern = /\/collage[\/\?](?:id=)?(\d+)/i
  const match = input.match(urlPattern)

  if (match && match[1]) {
    return parseInt(match[1], 10)
  }

  if (/^\d+$/.test(input.trim())) {
    return parseInt(input.trim(), 10)
  }

  return null
}

const currentSearchIndex = ref<number | null>(null)

const searchCollagesMethod = async (query: string): Promise<CollageSearchResult[]> => {
  const collageId = extractCollageIdFromUrl(query)

  if (collageId && currentSearchIndex.value !== null) {
    try {
      const collageData = await getCollage(collageId)
      const collageResult: CollageSearchResult = {
        id: collageData.collage.id,
        name: collageData.collage.name,
      } as CollageSearchResult

      onCollageSelected(collageResult, currentSearchIndex.value)

      return []
    } catch (error) {
      console.error('Failed to fetch collage by ID:', error)
      return []
    }
  }

  const searchQuery: SearchCollagesQuery = {
    name: query,
    page: 1,
    page_size: 10,
    tags: [],
  }

  const results = await searchCollages(searchQuery)
  return results.results || []
}

const onCollageSelected = (collage: CollageSearchResult, index: number) => {
  selectedCollages.value[index] = collage
  newCollageEntries.value[index].collage_id = collage.id

  if (noteInputRefs.value[index]?.value) {
    setTimeout(() => {
      noteInputRefs.value[index]?.value?.focus()
    }, 100)
  }
}

const clearCollageSelection = (index: number) => {
  selectedCollages.value[index] = null
  newCollageEntries.value[index].collage_id = 0
}

const sendCollageEntries = async () => {
  const hasEmptyCollage = newCollageEntries.value.some((entry) => entry.collage_id === 0)
  if (hasEmptyCollage) {
    console.error('Please select a collage for all entries')
    return
  }

  loading.value = true

  newCollageEntries.value.forEach((entry) => {
    switch (props.entryType) {
      case 'TitleGroup': {
        entry.title_group_id = props.entryId
        break
      }
      case 'Artist': {
        entry.artist_id = props.entryId
        break
      }
    }
  })

  createCollageEntries(newCollageEntries.value)
    .then((data) => {
      emit('addedEntries', data)
    })
    .finally(() => {
      loading.value = false
    })
}

const addCollageEntry = () => {
  selectedCollages.value.push(null)
  newCollageEntries.value.push({ collage_id: 0, note: null })
  noteInputRefs.value.push(ref(null))
}

const removeCollageEntry = (index: number) => {
  newCollageEntries.value.splice(index, 1)
  selectedCollages.value.splice(index, 1)
  noteInputRefs.value.splice(index, 1)
}

onMounted(() => addCollageEntry())
</script>

<style scoped>
#add-entries-to-collage-dialog {
  width: 70vw;
  display: flex;
  flex-direction: column;
  align-items: center;
}

.entries {
  width: 100%;
}

.entry {
  display: flex;
  width: 100%;
  gap: 5px;
  margin-bottom: 8px;
  align-items: flex-start;
}

.inputs-container {
  display: flex;
  gap: 5px;
  flex: 1;
  min-width: 0;
  max-width: 80%;
  margin-left: 5%;
}

.buttons-container {
  display: flex;
  gap: 5px;
  flex-shrink: 0;
  min-width: 70px;
  max-width: 20%;
}

.collage-search-wrapper,
.note {
  flex: 0 0 calc(50% - 2.5px);
  min-width: 0;
}

.collage-search-wrapper {
  display: flex;
}

.collage-search-wrapper :deep(.search-bar-generic) {
  width: 100%;
}

/* Selected collage display */
.selected-collage {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 12px;
  border: 1px solid var(--color-secondary);
  border-radius: 4px;
  height: 36px;
  width: 100%;
  box-sizing: border-box;
}

.collage-name {
  font-weight: 500;
  color: var(--vt-c-text-dark-2);

  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  flex: 1;
}

.clear-btn {
  flex-shrink: 0;
  margin-left: 8px;
}

.collage-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: 100%;
}

.collage-id {
  color: #6c757d;
  font-size: 0.875rem;
  margin-left: 8px;
  flex-shrink: 0;
}
</style>
