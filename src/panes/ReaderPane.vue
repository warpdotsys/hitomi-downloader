<script setup lang="ts">
import { useStore } from '../store.ts'
import { commands } from '../bindings.ts'
import { ref, watch, onMounted, onUnmounted, computed, nextTick } from 'vue'
import { PhCaretLeft, PhCaretRight, PhList, PhRows, PhArrowsOutSimple } from '@phosphor-icons/vue'
import { useI18n } from '../utils.ts'

const { t } = useI18n()
const store = useStore()

type ReadMode = 'page' | 'scroll'

const currentIndex = ref(0)
const currentImage = ref<string | undefined>(undefined)
const loading = ref(false)
const readMode = ref<ReadMode>('page')
const thumbnailsShowing = ref(false)
const fitWidth = ref(false)
const scrollContainerRef = ref<HTMLElement>()
const scrollImages = ref<Map<number, string>>(new Map())

const imageCache = new Map<number, Promise<string | undefined>>()
const thumbnailCache = new Map<number, string>()

const comic = computed(() => store.pickedComic)
const totalFiles = computed(() => comic.value?.files.length ?? 0)

function clearCache() {
  for (const promise of imageCache.values()) {
    promise.then(url => { if (url) URL.revokeObjectURL(url) }).catch(() => {})
  }
  imageCache.clear()
  scrollImages.value.clear()
  for (const url of thumbnailCache.values()) {
    URL.revokeObjectURL(url)
  }
  thumbnailCache.clear()
}

function getOrFetchImage(index: number): Promise<string | undefined> {
  if (!comic.value || index < 0 || index >= comic.value.files.length) {
    return Promise.resolve(undefined)
  }
  if (imageCache.has(index)) {
    return imageCache.get(index)!
  }

  const promise = (async () => {
    const c = comic.value
    if (!c) return undefined
    const file = c.files[index]
    const result = await commands.getImageData(c.id, c.isDownloaded ?? null, c.comicDownloadDir ?? null, file)
    if (result.status === 'ok') {
      const blob = new Blob([new Uint8Array(result.data)])
      return URL.createObjectURL(blob)
    } else {
      console.error(result.error)
      imageCache.delete(index)
      return undefined
    }
  })()

  imageCache.set(index, promise)
  return promise
}

async function loadThumbnail(index: number) {
  if (thumbnailCache.has(index) || !comic.value) return
  const url = await getOrFetchImage(index)
  if (url) thumbnailCache.set(index, url)
}

async function loadCurrentImage() {
  if (!comic.value || comic.value.files.length === 0) return
  loading.value = true
  const index = currentIndex.value

  const url = await getOrFetchImage(index)
  if (currentIndex.value !== index || store.pickedComic !== comic.value) return

  if (store.currentTabName === 'reader') {
    currentImage.value = url
  }
  loading.value = false

  // Preload adjacent images
  for (const i of [index + 1, index + 2, index - 1]) {
    if (i >= 0 && i < comic.value.files.length) {
      getOrFetchImage(i).catch(() => {})
    }
  }

  // Cleanup distant cache entries
  for (const key of imageCache.keys()) {
    if (key < index - 5 || key > index + 5) {
      const promise = imageCache.get(key)
      if (promise) {
        promise.then(u => { if (u) URL.revokeObjectURL(u) }).catch(() => {})
      }
      imageCache.delete(key)
    }
  }
}

async function loadScrollImages() {
  if (!comic.value) return
  const start = Math.max(0, currentIndex.value - 2)
  const end = Math.min(comic.value.files.length, currentIndex.value + 8)

  for (let i = start; i < end; i++) {
    if (!scrollImages.value.has(i)) {
      const url = await getOrFetchImage(i)
      if (url && comic.value === store.pickedComic) {
        scrollImages.value.set(i, url)
      }
    }
  }
}

function goToPage(index: number) {
  if (!comic.value) return
  const clamped = Math.max(0, Math.min(index, comic.value.files.length - 1))
  currentIndex.value = clamped

  if (readMode.value === 'scroll') {
    nextTick(() => {
      const el = document.getElementById(`scroll-page-${clamped}`)
      el?.scrollIntoView({ behavior: 'smooth', block: 'start' })
    })
  }
}

function prevPage() {
  if (currentIndex.value > 0) currentIndex.value--
}

function nextPage() {
  if (comic.value && currentIndex.value < comic.value.files.length - 1) currentIndex.value++
}

function toggleMode() {
  readMode.value = readMode.value === 'page' ? 'scroll' : 'page'
}

function handleKeydown(e: KeyboardEvent) {
  if (store.currentTabName !== 'reader') return

  switch (e.key) {
    case 'ArrowRight':
    case 'ArrowDown':
    case ' ':
      e.preventDefault()
      if (readMode.value === 'page') nextPage()
      break
    case 'ArrowLeft':
    case 'ArrowUp':
      e.preventDefault()
      if (readMode.value === 'page') prevPage()
      break
    case 'Home':
      e.preventDefault()
      goToPage(0)
      break
    case 'End':
      e.preventDefault()
      goToPage((comic.value?.files.length ?? 1) - 1)
      break
    case 'PageDown':
      e.preventDefault()
      goToPage(currentIndex.value + 10)
      break
    case 'PageUp':
      e.preventDefault()
      goToPage(currentIndex.value - 10)
      break
    case 't':
      thumbnailsShowing.value = !thumbnailsShowing.value
      break
    case 'f':
      fitWidth.value = !fitWidth.value
      break
    case 'm':
      toggleMode()
      break
  }
}

function handleScroll(e: Event) {
  if (readMode.value !== 'scroll' || !comic.value) return
  const container = e.target as HTMLElement
  const scrollTop = container.scrollTop
  const pageHeight = container.scrollHeight / comic.value.files.length
  const newIndex = Math.round(scrollTop / pageHeight)
  if (newIndex !== currentIndex.value && newIndex >= 0 && newIndex < comic.value.files.length) {
    currentIndex.value = newIndex
  }
}

watch(
  () => store.pickedComic,
  () => {
    clearCache()
    currentImage.value = undefined
    scrollImages.value.clear()
    currentIndex.value = 0
    if (store.currentTabName === 'reader') {
      if (readMode.value === 'page') loadCurrentImage()
      else loadScrollImages()
    }
  },
)

watch(
  () => store.currentTabName,
  (name) => {
    if (name === 'reader') {
      if (readMode.value === 'page') loadCurrentImage()
      else loadScrollImages()
    }
  },
)

watch(currentIndex, () => {
  if (readMode.value === 'page') loadCurrentImage()
  else loadScrollImages()

  // Load thumbnails for visible range
  if (thumbnailsShowing.value && comic.value) {
    const start = Math.max(0, currentIndex.value - 5)
    const end = Math.min(comic.value.files.length, currentIndex.value + 15)
    for (let i = start; i < end; i++) {
      loadThumbnail(i)
    }
  }
})

watch(readMode, () => {
  if (readMode.value === 'scroll') {
    loadScrollImages()
    nextTick(() => goToPage(currentIndex.value))
  }
})

watch(thumbnailsShowing, (showing) => {
  if (showing && comic.value) {
    const start = Math.max(0, currentIndex.value - 5)
    const end = Math.min(comic.value.files.length, currentIndex.value + 15)
    for (let i = start; i < end; i++) {
      loadThumbnail(i)
    }
  }
})

onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  clearCache()
  currentImage.value = undefined
  scrollImages.value.clear()
})
</script>

<template>
  <div class="h-full flex flex-col bg-gray-900 relative outline-none" tabindex="0">
    <!-- Toolbar -->
    <div class="flex items-center gap-2 px-3 py-1.5 bg-gray-800 text-white text-sm z-10">
      <n-button size="tiny" quaternary @click="prevPage" :disabled="currentIndex <= 0">
        <template #icon><n-icon><PhCaretLeft /></n-icon></template>
      </n-button>
      <span class="min-w-[80px] text-center">{{ currentIndex + 1 }} / {{ totalFiles }}</span>
      <n-button size="tiny" quaternary @click="nextPage" :disabled="currentIndex >= totalFiles - 1">
        <template #icon><n-icon><PhCaretRight /></n-icon></template>
      </n-button>
      <n-slider
        class="flex-1 mx-2"
        :value="currentIndex"
        :max="Math.max(0, totalFiles - 1)"
        :step="1"
        :tooltip="false"
        @update:value="goToPage" />
      <n-button size="tiny" quaternary :type="readMode === 'scroll' ? 'primary' : 'default'" @click="toggleMode" :title="t('reader_pane.toggle_mode')">
        <template #icon><n-icon><PhRows v-if="readMode === 'page'" /><PhList v-else /></n-icon></template>
      </n-button>
      <n-button size="tiny" quaternary :type="fitWidth ? 'primary' : 'default'" @click="fitWidth = !fitWidth" :title="t('reader_pane.fit_width')">
        <template #icon><n-icon><PhArrowsOutSimple /></n-icon></template>
      </n-button>
      <n-button size="tiny" quaternary :type="thumbnailsShowing ? 'primary' : 'default'" @click="thumbnailsShowing = !thumbnailsShowing" :title="t('reader_pane.thumbnails')">
        <template #icon><n-icon><PhList /></n-icon></template>
      </n-button>
    </div>

    <!-- Page mode -->
    <div v-if="readMode === 'page'" class="flex-1 flex items-center justify-center overflow-hidden p-2">
      <n-spin v-if="loading" class="absolute z-10" />
      <img
        v-if="currentImage"
        :src="currentImage"
        :class="['object-contain transition-all duration-200', fitWidth ? 'w-full h-auto' : 'max-w-full max-h-full']"
        @click="nextPage" />
    </div>

    <!-- Scroll mode -->
    <div
      v-else
      ref="scrollContainerRef"
      class="flex-1 overflow-auto"
      @scroll="handleScroll">
      <div class="flex flex-col items-center gap-1 p-2">
        <template v-for="i in totalFiles" :key="i - 1">
          <img
            :id="`scroll-page-${i - 1}`"
            v-if="scrollImages.has(i - 1)"
            :src="scrollImages.get(i - 1)"
            :class="['transition-all duration-200', fitWidth ? 'w-full' : 'max-w-full']"
            :alt="`Page ${i}`" />
          <div v-else class="w-full h-[600px] flex items-center justify-center bg-gray-800">
            <n-spin size="small" />
          </div>
        </template>
      </div>
    </div>

    <!-- Thumbnail strip -->
    <Transition name="slide-up">
      <div v-if="thumbnailsShowing && comic" class="h-24 bg-gray-800 border-t border-gray-700 overflow-x-auto flex gap-1 p-2">
        <div
          v-for="i in totalFiles"
          :key="i - 1"
          :class="[
            'h-full flex-shrink-0 w-16 cursor-pointer rounded overflow-hidden border-2 transition-all duration-150',
            currentIndex === i - 1 ? 'border-blue-500 scale-110' : 'border-transparent hover:border-gray-500',
          ]"
          @click="goToPage(i - 1)">
          <img
            v-if="thumbnailCache.has(i - 1)"
            :src="thumbnailCache.get(i - 1)"
            class="w-full h-full object-cover" />
          <div v-else class="w-full h-full flex items-center justify-center text-gray-400 text-xs">
            {{ i }}
          </div>
        </div>
      </div>
    </Transition>
  </div>
</template>

<style scoped>
.slide-up-enter-active,
.slide-up-leave-active {
  transition: transform 0.2s ease, opacity 0.2s ease;
}
.slide-up-enter-from,
.slide-up-leave-to {
  transform: translateY(100%);
  opacity: 0;
}
</style>
