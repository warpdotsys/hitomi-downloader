<script setup lang="tsx">
import { computed, nextTick, ref, watch } from 'vue'
import { commands, Suggestion } from '../bindings.ts'
import { DropdownOption, SelectOption, useMessage, useNotification, NIcon } from 'naive-ui'
import ComicCard from '../components/ComicCard.vue'
import { useStore } from '../store.ts'
import { useI18n } from '../utils.ts'
import { PhMagnifyingGlass, PhArrowRight, PhChecks, PhCloudArrowDown } from '@phosphor-icons/vue'
import FloatLabelInput from '../components/FloatLabelInput.vue'
import { SelectionArea } from '@viselect/vue'
import { useMultiSelect } from '../composables/useMultiSelect.ts'


const { t } = useI18n()

const store = useStore()

const message = useMessage()
const notification = useNotification()

const searchInput = ref<string>('')
const searchInputRef = ref<InstanceType<typeof FloatLabelInput>>()
const comicIdInput = ref<string>('')
const currentPage = ref<number>(1)
const comicCardContainerRef = ref<HTMLElement>()
const {
  onSuggestionSelect,
  getSearchSuggestions,
  suggestionOptions,
  suggestionsShowing,
  dropdownFocused,
  dropdownX,
  dropdownY,
} = useSuggestion()

const searching = ref<boolean>(false)

watch(
  () => store.searchResult,
  () => {
    if (comicCardContainerRef.value !== undefined) {
      comicCardContainerRef.value.scrollTo({ top: 0, behavior: 'instant' })
    }
  },
)

async function search(query: string, pageNum: number) {
  if (searching.value) {
    message.warning(() => t('search_pane.searching_warning'))
    return
  }

  searchInput.value = query
  currentPage.value = pageNum
  store.currentTabName = 'search'
  searching.value = true
  suggestionsShowing.value = false

  // TODO: support sort by popularity
  const result = await commands.search(query, pageNum, false)
  if (result.status === 'error') {
    console.error(result.error)
    searching.value = false
    return
  }
  store.searchResult = result.data

  searching.value = false
}

async function handlePageChange(pageNum: number) {
  if (store.searchResult === undefined) {
    return
  }

  currentPage.value = pageNum

  const result = await commands.getPage(store.searchResult.ids, pageNum)
  if (result.status === 'error') {
    console.error(result.error)
    return
  }

  store.searchResult = result.data
}

function getComicIdFromComicIdInput(): number | undefined {
  const comicIdString = comicIdInput.value.trim()
  // if it is a number, return it directly
  const comicId = parseInt(comicIdString)
  if (!isNaN(comicId)) {
    return comicId
  }
  // otherwise, extract it from the url
  const regex = /-(\d+).html/
  const match = comicIdString.match(regex)
  if (match === null || match[1] === null) {
    return
  }
  return parseInt(match[1])
}

async function pickComic() {
  const comicId = getComicIdFromComicIdInput()
  if (comicId === undefined) {
    notification.error({
      title: () => t('search_pane.comic_id_invalid'),
      description: () => t('search_pane.enter_comic_id_or_url'),
    })
    return
  }

  const result = await commands.getComic(comicId)
  if (result.status === 'error') {
    console.error(result.error)
    return
  }

  store.pickedComic = result.data
  store.currentTabName = 'comic'
}

function handleSearchInputKeydown(e: KeyboardEvent) {
  if (e.key === 'ArrowUp' || e.key === 'ArrowDown') {
    dropdownFocused.value = suggestionsShowing.value
  } else if (e.key === 'Enter') {
    if (!dropdownFocused.value) {
      search(searchInput.value.trim(), 1)
    }
  } else if (e.key === 'ArrowLeft' || e.key === 'ArrowRight') {
    e.stopPropagation()
  }
}

function handleSearchInputKeyup(e: KeyboardEvent) {
  const { key } = e
  if (key !== 'Enter' && key !== 'ArrowUp' && key !== 'ArrowDown' && key !== 'ArrowLeft' && key !== 'ArrowRight') {
    getSearchSuggestions()
  }
}

function useSuggestion() {
  const suggestionsShowing = ref<boolean>(false)
  const suggestions = ref<Suggestion[]>([])
  const dropdownFocused = ref<boolean>(false)

  const dropdownX = computed<number>(() => {
    const x = searchInputRef.value?.NInputRef?.inputElRef?.getBoundingClientRect().left
    return x ? x - 2 : 0
  })

  const dropdownY = computed<number>(() => {
    const y = searchInputRef.value?.NInputRef?.inputElRef?.getBoundingClientRect().bottom
    return y ? y - 2 : 0
  })

  watch(suggestionsShowing, (showing) => {
    if (!showing) {
      dropdownFocused.value = false
    }
  })

  const suggestionOptions = computed<SelectOption[]>(() => {
    return suggestions.value.map((suggestion) => {
      const { s, n, t } = suggestion
      return {
        label: () => (
          <div class="flex">
            <span class="font-bold">{s}</span>
            <span class="text-gray-5 px-1">({n})</span>
            <span class="ml-auto text-gray-5">{t}</span>
          </div>
        ),
        key: `${n}:${s.replace(' ', '_')}`,
      }
    })
  })

  async function getSearchSuggestions() {
    suggestionsShowing.value = false
    let lastWord = searchInput.value.split(' ').pop()
    if (lastWord === undefined || lastWord === '') {
      suggestions.value = []
      return
    }

    if (lastWord.startsWith('-')) {
      lastWord = lastWord.substring(1)
    }

    const result = await commands.getSearchSuggestions(lastWord)
    if (result.status === 'error') {
      console.error(result.error)
      return
    }

    suggestions.value = result.data
    suggestionsShowing.value = true
  }

  async function onSuggestionSelect(key: string) {
    const words = searchInput.value.split(' ')
    const isNegative = words.pop()?.startsWith('-')
    if (isNegative === true) {
      words.push(`-${key}`)
    } else {
      words.push(key)
    }

    searchInput.value = words.join(' ') + ' '
    suggestionsShowing.value = false

    await nextTick()
    searchInputRef.value?.NInputRef?.scrollTo({
      behavior: 'smooth',
      left: 10000,
    })
  }

  return {
    suggestionsShowing,
    suggestionOptions,
    getSearchSuggestions,
    onSuggestionSelect,
    dropdownFocused,
    dropdownX,
    dropdownY,
  }
}

const { selectedIds, selectionAreaRef, selectableRefs, updateSelectedIds, unselectAll, onContextMenu, toggleSelection } = useMultiSelect()
const { contextMenuX, contextMenuY, contextMenuShowing, contextMenuOptions, showContextMenu } = useContextMenu()

watch(() => store.searchResult, () => {
  selectedIds.value.clear()
})



function useContextMenu() {
  const contextMenuX = ref<number>(0)
  const contextMenuY = ref<number>(0)
  const contextMenuShowing = ref<boolean>(false)
  const contextMenuOptions: DropdownOption[] = [
    {
      label: t('common.select_all'),
      key: 'select-all',
      icon: () => (
        <n-icon size="20">
          <PhChecks />
        </n-icon>
      ),
      props: {
        onClick: () => {
          if (selectionAreaRef.value === undefined) {
            return
          }
          const selection = selectionAreaRef.value.selection
          if (selection === undefined) {
            return
          }
          selection.select(selectableRefs.value)
          contextMenuShowing.value = false
        },
      },
    },
    {
      label: t('download_button.quick_download'),
      key: 'download',
      icon: () => (
        <n-icon size="20">
          <PhCloudArrowDown />
        </n-icon>
      ),
      props: {
        onClick: () => {
          ;(store.searchResult?.comics ?? [])
            .filter((comic) => selectedIds.value.has(comic.id))
            .forEach(async (comic) => {
              const result = await commands.createDownloadTask(comic)
              if (result.status === 'error') {
                console.error(result.error)
              }
            })
          contextMenuShowing.value = false
        },
      },
    },
  ]

  async function showContextMenu(e: MouseEvent) {
    contextMenuShowing.value = false
    await nextTick()
    contextMenuShowing.value = true
    contextMenuX.value = e.clientX
    contextMenuY.value = e.clientY
  }

  return {
    contextMenuX,
    contextMenuY,
    contextMenuShowing,
    contextMenuOptions,
    showContextMenu,
  }
}

defineExpose({ search })
</script>

<template>
  <div class="h-full flex flex-col gap-2">
    <n-input-group class="box-border px-2 pt-2">
      <FloatLabelInput
        :label="t('search_pane.search_by_query')"
        ref="searchInputRef"
        size="small"
        v-model:value="searchInput"
        clearable
        @keyup="handleSearchInputKeyup"
        @keydown="handleSearchInputKeydown" />
      <n-dropdown
        v-if="suggestionsShowing"
        :x="dropdownX"
        :y="dropdownY"
        size="small"
        :show="suggestionsShowing"
        :options="suggestionOptions"
        @select="onSuggestionSelect"
        @clickoutside="() => (suggestionsShowing = false)" />
      <n-button :loading="searching" type="primary" size="small" @click="search(searchInput.trim(), 1)">
        <template #icon>
          <n-icon size="22">
            <PhMagnifyingGlass />
          </n-icon>
        </template>
      </n-button>
    </n-input-group>

    <n-input-group class="box-border px-2">
      <FloatLabelInput
        :label="t('search_pane.search_by_id')"
        class="text-align-left"
        size="small"
        v-model:value="comicIdInput"
        clearable
        @keydown.enter="pickComic" />
      <n-button type="primary" size="small" @click="pickComic">
        <template #icon>
          <n-icon size="22">
            <PhArrowRight />
          </n-icon>
        </template>
      </n-button>
    </n-input-group>

    <SelectionArea
      v-if="store.searchResult !== undefined"
      ref="selectionAreaRef"
      class="flex flex-col gap-row-2 overflow-auto box-border px-2 selection-container"
      :options="{ selectables: '.selectable', features: { deselectOnBlur: true } }"
      @contextmenu="showContextMenu"
      @move="updateSelectedIds"
      @start="unselectAll">
      <div
        v-for="(comic, index) in store.searchResult.comics"
        :key="comic.id"
        ref="selectableRefs"
        :data-key="comic.id"
        :class="['selectable rounded p-[2px] relative', selectedIds.has(comic.id) ? 'selected' : '']"
        @contextmenu="() => onContextMenu(comic.id)">
        <comic-card
          :search="search"
          v-model:comic="store.searchResult.comics[index]" />
        <div class="absolute top-2 left-2 z-10 bg-white/50 rounded flex items-center justify-center p-1" @mousedown.stop @touchstart.stop @pointerdown.stop @click.stop>
          <n-checkbox
            size="large"
            :checked="selectedIds.has(comic.id)"
            @update:checked="(checked: boolean) => toggleSelection(comic.id, checked)"
          />
        </div>
      </div>
      <n-dropdown
        placement="bottom-start"
        trigger="manual"
        :x="contextMenuX"
        :y="contextMenuY"
        :options="contextMenuOptions"
        :show="contextMenuShowing"
        :on-clickoutside="() => (contextMenuShowing = false)" />
    </SelectionArea>

    <n-pagination
      v-if="store.searchResult !== undefined"
      class="box-border p-2 pt-0 mt-auto"
      :page-count="store.searchResult.totalPage"
      :page="currentPage"
      @update:page="handlePageChange" />
  </div>
</template>
