<script setup lang="tsx">
import { ProgressData } from '../types.ts'
import { computed, nextTick, ref, watch } from 'vue'
import { useStore } from '../store.ts'
import ComicCard from './ComicCard.vue'
import { SelectionArea } from '@viselect/vue'
import { useMultiSelect } from '../composables/useMultiSelect.ts'
import { DropdownOption } from 'naive-ui'
import { commands } from '../bindings.ts'
import { PhChecks, PhArrowClockwise } from '@phosphor-icons/vue'
import { useI18n } from '../utils.ts'

const { t } = useI18n()
const store = useStore()

defineProps<{
  search: (query: string, pageNum: number) => Promise<void>
}>()

const completedProgresses = computed<[number, ProgressData][]>(() =>
  Array.from(store.progresses.entries())
    .filter(([, { state }]) => state === 'Completed')
    .sort((a, b) => {
      return b[1].totalImgCount - a[1].totalImgCount
    }),
)


const { selectedIds, selectionAreaRef, selectableRefs, updateSelectedIds, unselectAll, onContextMenu } = useMultiSelect()
const { dropdownX, dropdownY, dropdownShowing, dropdownOptions, showDropdown } = useDropdown()

watch(completedProgresses, () => {
  const completedIds = new Set(completedProgresses.value.map(([chapterId]) => chapterId))
  // only retain selected ids that are completed
  selectedIds.value = new Set([...selectedIds.value].filter((comicId) => completedIds.has(comicId)))
})


function useDropdown() {
  const dropdownX = ref<number>(0)
  const dropdownY = ref<number>(0)
  const dropdownShowing = ref<boolean>(false)
  const dropdownOptions: DropdownOption[] = [
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
          dropdownShowing.value = false
        },
      },
    },
    {
      label: t('download_button.download_again'),
      key: 'redownload',
      icon: () => (
        <n-icon size="20">
          <PhArrowClockwise />
        </n-icon>
      ),
      props: {
        onClick: () => {
          completedProgresses.value
            .filter(([id]) => selectedIds.value.has(id))
            .forEach(async ([, progressData]) => {
              const result = await commands.createDownloadTask(progressData.comic)
              if (result.status === 'error') {
                console.error(result.error)
              }
            })
          dropdownShowing.value = false
        },
      },
    },
  ]

  async function showDropdown(e: MouseEvent) {
    dropdownShowing.value = false
    await nextTick()
    dropdownShowing.value = true
    dropdownX.value = e.clientX
    dropdownY.value = e.clientY
  }

  return {
    dropdownX,
    dropdownY,
    dropdownShowing,
    dropdownOptions,
    showDropdown,
  }
}
</script>

<template>
  <SelectionArea
    ref="selectionAreaRef"
    class="h-full flex flex-col overflow-auto px-2 gap-2 selection-container"
    :options="{ selectables: '.selectable', features: { deselectOnBlur: true } }"
    @contextmenu="showDropdown"
    @move="updateSelectedIds"
    @start="unselectAll">
    <div
      v-for="[comicId, progressData] in completedProgresses"
      :key="comicId"
      ref="selectableRefs"
      :data-key="comicId"
      :class="['selectable rounded p-[2px]', selectedIds.has(comicId) ? 'selected' : '']"
      @contextmenu="() => onContextMenu(comicId)">
      <comic-card :search="search" :comic="progressData.comic" show-export />
    </div>
    <n-dropdown
      placement="bottom-start"
      trigger="manual"
      :x="dropdownX"
      :y="dropdownY"
      :options="dropdownOptions"
      :show="dropdownShowing"
      :on-clickoutside="() => (dropdownShowing = false)" />
  </SelectionArea>
</template>
