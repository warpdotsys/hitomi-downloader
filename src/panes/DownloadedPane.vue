<script setup lang="tsx">
import { Comic, commands, events } from '../bindings.ts'
import { computed, onMounted, ref, watch, nextTick } from 'vue'
import { MessageReactive, useMessage, DropdownOption, NIcon } from 'naive-ui'
import { open } from '@tauri-apps/plugin-dialog'
import { useStore } from '../store.ts'
import ComicCard from '../components/ComicCard.vue'
import { useI18n } from '../utils.ts'
import { PhFolderOpen, PhChecks, PhFilePdf, PhFileZip, PhDownload } from '@phosphor-icons/vue'
import { SelectionArea } from '@viselect/vue'
import { useMultiSelect } from '../composables/useMultiSelect.ts'

const { t } = useI18n()

interface ProgressData {
  title: string
  progressMessage: MessageReactive
}

defineProps<{
  search: (query: string, pageNum: number) => Promise<void>
}>()

const store = useStore()

const message = useMessage()

const comicCardContainerRef = ref<HTMLElement>()

const PAGE_SIZE = 20

const downloadedComics = ref<Comic[]>([])
const currentPage = ref<number>(1)
const pageCount = computed<number>(() => {
  return Math.ceil(downloadedComics.value.length / PAGE_SIZE)
})
// currentPageComics is a computed property that returns the comics on the current page
const currentPageComics = computed<Comic[]>(() => {
  const start = (currentPage.value - 1) * PAGE_SIZE
  const end = start + PAGE_SIZE
  return downloadedComics.value.slice(start, end)
})

watch(currentPage, () => {
  if (comicCardContainerRef.value !== undefined) {
    comicCardContainerRef.value.scrollTo({ top: 0, behavior: 'instant' })
  }
})

// listen for changes in the tab and update the list of downloaded comics
watch(
  () => store.currentTabName,
  async () => {
    if (store.currentTabName !== 'downloaded') {
      return
    }

    downloadedComics.value = await commands.getDownloadedComics()
  },
  { immediate: true },
)

useProgressTracking()

function useProgressTracking() {
  const progresses = new Map<string, ProgressData>()

  async function handleExportPdfEvents() {
    await events.exportPdfEvent.listen(async ({ payload: exportEvent }) => {
      if (exportEvent.event === 'Start') {
        const { uuid, title } = exportEvent.data
        createProgress(uuid, title, t('downloaded_pane.pdf_exporting'))
      } else if (exportEvent.event === 'Error') {
        errorProgress(exportEvent.data.uuid, t('downloaded_pane.pdf_export_error'))
      } else if (exportEvent.event === 'End') {
        completeProgress(exportEvent.data.uuid, t('downloaded_pane.pdf_exported'))
      }
    })
  }

  async function handleExportCbzEvents() {
    await events.exportCbzEvent.listen(async ({ payload: exportEvent }) => {
      if (exportEvent.event === 'Start') {
        const { uuid, title } = exportEvent.data
        createProgress(uuid, title, t('downloaded_pane.cbz_exporting'))
      } else if (exportEvent.event === 'Error') {
        errorProgress(exportEvent.data.uuid, t('downloaded_pane.cbz_export_error'))
      } else if (exportEvent.event === 'End') {
        completeProgress(exportEvent.data.uuid, t('downloaded_pane.cbz_exported'))
      }
    })
  }

  // Create progress message
  function createProgress(uuid: string, title: string, actionMessage: string) {
    progresses.set(uuid, {
      title,
      progressMessage: message.loading(
        () => {
          const progressData = progresses.get(uuid)
          if (progressData === undefined) return ''
          return `${progressData.title} ${actionMessage}`
        },
        { duration: 0 },
      ),
    })
  }

  function errorProgress(uuid: string, actionMessage: string) {
    const progressData = progresses.get(uuid)
    if (progressData) {
      progressData.progressMessage.type = 'error'
      progressData.progressMessage.content = `${progressData.title} ${actionMessage}`
      setTimeout(() => {
        progressData.progressMessage.destroy()
        progresses.delete(uuid)
      }, 3000)
    }
  }

  // Set the progress message as complete
  function completeProgress(uuid: string, actionMessage: string) {
    const progressData = progresses.get(uuid)
    if (progressData) {
      progressData.progressMessage.type = 'success'
      progressData.progressMessage.content = `${progressData.title} ${actionMessage}`
      setTimeout(() => {
        progressData.progressMessage.destroy()
        progresses.delete(uuid)
      }, 3000)
    }
  }

  onMounted(async () => {
    await handleExportPdfEvents()
    await handleExportCbzEvents()
  })
}

async function selectExportDir() {
  if (store.config === undefined) {
    return
  }

  const selectedDirPath = await open({ directory: true })
  if (selectedDirPath === null) {
    return
  }
  store.config.exportDir = selectedDirPath
}

async function showExportDirInFileManager() {
  if (store.config === undefined) {
    return
  }
  const result = await commands.showPathInFileManager(store.config.exportDir)
  if (result.status === 'error') {
    console.error(result.error)
  }
}

async function exportAllCbz() {
  const comics = downloadedComics.value
  if (comics.length === 0) return
  message.info(t('downloaded_pane.export_all_cbz_started', { count: comics.length }))
  for (const comic of comics) {
    const result = await commands.exportCbz(comic)
    if (result.status === 'error') {
      console.error(result.error)
    }
  }
}

async function exportAllPdf() {
  const comics = downloadedComics.value
  if (comics.length === 0) return
  message.info(t('downloaded_pane.export_all_pdf_started', { count: comics.length }))
  for (const comic of comics) {
    const result = await commands.exportPdf(comic)
    if (result.status === 'error') {
      console.error(result.error)
    }
  }
}

const { selectedIds, selectionAreaRef, selectableRefs, updateSelectedIds, unselectAll, onContextMenu, toggleSelection } = useMultiSelect()
const { contextMenuX, contextMenuY, contextMenuShowing, contextMenuOptions, showContextMenu } = useContextMenu()

watch(currentPageComics, () => {
  const downloadedIds = new Set(currentPageComics.value.map(comic => comic.id))
  // only retain selected ids that are current page
  selectedIds.value = new Set([...selectedIds.value].filter((comicId) => downloadedIds.has(comicId)))
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
      label: t('common.export_pdf'),
      key: 'export-pdf',
      icon: () => (
        <n-icon size="20">
          <PhFilePdf />
        </n-icon>
      ),
      props: {
        onClick: () => {
          currentPageComics.value
            .filter((comic) => selectedIds.value.has(comic.id))
            .forEach(async (comic) => {
              const result = await commands.exportPdf(comic)
              if (result.status === 'error') {
                console.error(result.error)
              }
            })
          contextMenuShowing.value = false
        },
      },
    },
    {
      label: t('common.export_cbz'),
      key: 'export-cbz',
      icon: () => (
        <n-icon size="20">
          <PhFileZip />
        </n-icon>
      ),
      props: {
        onClick: () => {
          currentPageComics.value
            .filter((comic) => selectedIds.value.has(comic.id))
            .forEach(async (comic) => {
              const result = await commands.exportCbz(comic)
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
</script>

<template>
  <div v-if="store.config !== undefined" class="h-full flex flex-col gap-2">
    <n-input-group class="box-border px-2 pt-2">
      <n-input-group-label size="small">
        {{ t('common.export_directory') }}
      </n-input-group-label>
      <n-input v-model:value="store.config.exportDir" size="small" readonly @click="selectExportDir" />
      <n-button class="w-9" size="small" @click="showExportDirInFileManager">
        <template #icon>
          <n-icon size="20">
            <PhFolderOpen />
          </n-icon>
        </template>
      </n-button>
    </n-input-group>
    <div class="flex gap-2 px-2">
      <n-button size="small" :disabled="downloadedComics.length === 0" @click="exportAllCbz">
        <template #icon>
          <n-icon size="18"><PhFileZip /></n-icon>
        </template>
        {{ t('downloaded_pane.export_all_cbz') }} ({{ downloadedComics.length }})
      </n-button>
      <n-button size="small" :disabled="downloadedComics.length === 0" @click="exportAllPdf">
        <template #icon>
          <n-icon size="18"><PhFilePdf /></n-icon>
        </template>
        {{ t('downloaded_pane.export_all_pdf') }} ({{ downloadedComics.length }})
      </n-button>
    </div>

    <SelectionArea
      v-if="currentPageComics.length > 0"
      ref="selectionAreaRef"
      class="flex flex-col gap-row-2 overflow-auto box-border px-2 selection-container"
      :options="{ selectables: '.selectable', features: { deselectOnBlur: true } }"
      @contextmenu="showContextMenu"
      @move="updateSelectedIds"
      @start="unselectAll">
      <div
        v-for="comic in currentPageComics"
        :key="comic.id"
        ref="selectableRefs"
        :data-key="comic.id"
        :class="['selectable rounded p-[2px] relative', selectedIds.has(comic.id) ? 'selected' : '']"
        @contextmenu="() => onContextMenu(comic.id)">
        <comic-card :search="search" :comic="comic" show-export />
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
    <div v-else ref="comicCardContainerRef" class="flex-1"></div>

    <n-pagination
      class="box-border p-2 pt-0 mt-auto"
      :page-count="pageCount"
      :page="currentPage"
      @update:page="currentPage = $event" />
  </div>
</template>
