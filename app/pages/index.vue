<script setup lang="ts">
interface FileItem {
  id: string
  file: File
  name: string
  size: number
  type: 'pdf' | 'image'
  pageCount?: number
  previewUrl?: string
}

const fileInput = ref<HTMLInputElement>()
const files = ref<FileItem[]>([])
const isDragOver = ref(false)
const isMerging = ref(false)
const mergeError = ref<string | null>(null)

const dragReorderIndex = ref(-1)
const dragOverIndex = ref(-1)

const canMerge = computed(() => files.value.length >= 1 && !isMerging.value)

const totalPages = computed(() =>
  files.value.reduce((sum, f) => sum + (f.pageCount ?? 1), 0)
)

const mergeLabel = computed(() => {
  if (isMerging.value) return 'Merging…'
  if (files.value.length === 1) return 'Convert to PDF'
  return 'Merge & Download PDF'
})

function isValidType(file: File) {
  return ['application/pdf', 'image/jpeg', 'image/png', 'image/webp'].includes(file.type)
}

function formatSize(bytes: number) {
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(0)} KB`
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
}

async function getPdfPageCount(file: File): Promise<number> {
  try {
    const { PDFDocument } = await import('pdf-lib')
    const bytes = await file.arrayBuffer()
    const pdf = await PDFDocument.load(bytes, { ignoreEncryption: true })
    return pdf.getPageCount()
  } catch {
    return 1
  }
}

async function processFiles(incoming: File[]) {
  for (const file of incoming) {
    if (!isValidType(file)) continue
    if (files.value.some(f => f.name === file.name && f.size === file.size)) continue

    const item: FileItem = {
      id: crypto.randomUUID(),
      file,
      name: file.name,
      size: file.size,
      type: file.type === 'application/pdf' ? 'pdf' : 'image'
    }

    if (item.type === 'image') {
      item.previewUrl = URL.createObjectURL(file)
    }

    files.value.push(item)

    if (item.type === 'pdf') {
      item.pageCount = await getPdfPageCount(file)
    }
  }
}

function onDropZoneDrop(e: DragEvent) {
  isDragOver.value = false
  const dropped = Array.from(e.dataTransfer?.files ?? [])
  if (dropped.length) processFiles(dropped)
}

function onDropZoneDragOver(e: DragEvent) {
  if (e.dataTransfer?.types.includes('Files')) {
    isDragOver.value = true
  }
}

function openFilePicker() {
  fileInput.value?.click()
}

function onFileInputChange(e: Event) {
  const input = e.target as HTMLInputElement
  if (input.files?.length) {
    processFiles(Array.from(input.files))
    input.value = ''
  }
}

function removeFile(id: string) {
  const idx = files.value.findIndex(f => f.id === id)
  if (idx === -1) return
  const item = files.value[idx]!
  if (item.previewUrl) URL.revokeObjectURL(item.previewUrl)
  files.value.splice(idx, 1)
}

function clearAll() {
  files.value.forEach((f) => {
    if (f.previewUrl) URL.revokeObjectURL(f.previewUrl)
  })
  files.value = []
  mergeError.value = null
}

function onItemDragStart(e: DragEvent, index: number) {
  dragReorderIndex.value = index
  e.dataTransfer!.effectAllowed = 'move'
  e.dataTransfer!.setData('text/plain', String(index))
}

function onItemDragOver(e: DragEvent, index: number) {
  if (dragReorderIndex.value === -1) return
  e.preventDefault()
  dragOverIndex.value = index
}

function onItemDrop(e: DragEvent, targetIndex: number) {
  e.preventDefault()
  const src = dragReorderIndex.value
  dragReorderIndex.value = -1
  dragOverIndex.value = -1
  if (src === -1 || src === targetIndex) return
  const [item] = files.value.splice(src, 1) as [FileItem]
  files.value.splice(targetIndex, 0, item)
}

function onItemDragEnd() {
  dragReorderIndex.value = -1
  dragOverIndex.value = -1
}

function convertImageToPng(file: File): Promise<Uint8Array> {
  return new Promise((resolve, reject) => {
    const img = new Image()
    const url = URL.createObjectURL(file)
    img.onload = () => {
      const canvas = document.createElement('canvas')
      canvas.width = img.naturalWidth
      canvas.height = img.naturalHeight
      canvas.getContext('2d')!.drawImage(img, 0, 0)
      URL.revokeObjectURL(url)
      canvas.toBlob(
        blob => blob
          ? blob.arrayBuffer().then(buf => resolve(new Uint8Array(buf)))
          : reject(new Error('Canvas conversion failed')),
        'image/png'
      )
    }
    img.onerror = () => {
      URL.revokeObjectURL(url)
      reject(new Error('Image load failed'))
    }
    img.src = url
  })
}

async function merge() {
  if (!canMerge.value) return
  mergeError.value = null
  isMerging.value = true

  try {
    const { PDFDocument } = await import('pdf-lib')
    const merged = await PDFDocument.create()

    for (const item of files.value) {
      if (item.type === 'pdf') {
        const bytes = new Uint8Array(await item.file.arrayBuffer())
        const pdf = await PDFDocument.load(bytes)
        const pages = await merged.copyPages(pdf, pdf.getPageIndices())
        pages.forEach(p => merged.addPage(p))
      } else {
        let imageBytes: Uint8Array
        let embedMethod: 'embedJpg' | 'embedPng'

        if (item.file.type === 'image/jpeg') {
          imageBytes = new Uint8Array(await item.file.arrayBuffer())
          embedMethod = 'embedJpg'
        } else {
          imageBytes = await convertImageToPng(item.file)
          embedMethod = 'embedPng'
        }

        const image = await merged[embedMethod](imageBytes)
        const { width, height } = image.scale(1)
        const page = merged.addPage([width, height])
        page.drawImage(image, { x: 0, y: 0, width, height })
      }
    }

    const bytes = await merged.save()
    const blob = new Blob([bytes.buffer as ArrayBuffer], { type: 'application/pdf' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url
    a.download = 'merged.pdf'
    document.body.appendChild(a)
    a.click()
    document.body.removeChild(a)
    setTimeout(() => URL.revokeObjectURL(url), 1000)
  } catch (e) {
    mergeError.value = e instanceof Error ? e.message : 'Failed to merge files. Please try again.'
  } finally {
    isMerging.value = false
  }
}

onUnmounted(() => {
  files.value.forEach((f) => {
    if (f.previewUrl) URL.revokeObjectURL(f.previewUrl)
  })
})
</script>

<template>
  <UContainer class="py-12 max-w-2xl">
    <div class="text-center mb-10">
      <h1 class="text-4xl font-bold tracking-tight text-gray-900 dark:text-white mb-3">
        PDF & Image Merger
      </h1>
      <p class="text-lg text-gray-500 dark:text-gray-400">
        Merge PDFs and images into a single PDF — entirely in your browser
      </p>
    </div>

    <!-- Drop Zone -->
    <div
      class="border-2 border-dashed rounded-xl p-12 text-center cursor-pointer transition-colors mb-6"
      :class="isDragOver
        ? 'border-primary-500 bg-primary-50 dark:bg-primary-950'
        : 'border-gray-300 dark:border-gray-700 hover:border-primary-400 dark:hover:border-primary-500'"
      @click="openFilePicker"
      @dragover.prevent="onDropZoneDragOver"
      @dragleave="isDragOver = false"
      @drop.prevent="onDropZoneDrop"
    >
      <UIcon
        name="i-lucide-upload-cloud"
        class="w-10 h-10 mx-auto mb-3 transition-colors"
        :class="isDragOver ? 'text-primary-500' : 'text-gray-400'"
      />
      <p class="text-base font-medium text-gray-700 dark:text-gray-300 mb-1">
        Drop files here or
        <span class="text-primary-600 dark:text-primary-400">click to browse</span>
      </p>
      <p class="text-sm text-gray-400 dark:text-gray-500">
        PDF, JPG, PNG, WebP
      </p>
    </div>

    <input
      ref="fileInput"
      type="file"
      multiple
      accept=".pdf,image/jpeg,image/png,image/webp"
      class="hidden"
      @change="onFileInputChange"
    >

    <!-- File List -->
    <div
      v-if="files.length > 0"
      class="mb-6"
    >
      <div class="flex items-center justify-between mb-3">
        <p class="text-sm text-gray-500 dark:text-gray-400">
          {{ files.length }} file{{ files.length === 1 ? '' : 's' }}
          <span v-if="totalPages > 0"> · {{ totalPages }} page{{ totalPages === 1 ? '' : 's' }}</span>
        </p>
        <UButton
          variant="ghost"
          size="xs"
          color="neutral"
          @click="clearAll"
        >
          Clear all
        </UButton>
      </div>

      <div class="space-y-2">
        <div
          v-for="(item, index) in files"
          :key="item.id"
          draggable="true"
          class="flex items-center gap-3 px-3 py-2.5 bg-white dark:bg-gray-900 border rounded-xl transition-all select-none"
          :class="{
            'border-primary-400 ring-2 ring-primary-200 dark:ring-primary-900': dragOverIndex === index && dragReorderIndex !== index,
            'opacity-30 scale-[0.98]': dragReorderIndex === index,
            'border-gray-200 dark:border-gray-800': dragOverIndex !== index || dragReorderIndex === index
          }"
          @dragstart="onItemDragStart($event, index)"
          @dragover="onItemDragOver($event, index)"
          @drop="onItemDrop($event, index)"
          @dragend="onItemDragEnd"
        >
          <UIcon
            name="i-lucide-grip-vertical"
            class="w-4 h-4 text-gray-300 dark:text-gray-600 flex-shrink-0 cursor-grab active:cursor-grabbing"
          />

          <div class="w-9 h-9 flex-shrink-0 rounded-lg overflow-hidden flex items-center justify-center bg-gray-100 dark:bg-gray-800">
            <img
              v-if="item.previewUrl"
              :src="item.previewUrl"
              :alt="item.name"
              class="w-full h-full object-cover"
            >
            <UIcon
              v-else
              name="i-lucide-file-text"
              class="w-5 h-5 text-red-500"
            />
          </div>

          <div class="flex-1 min-w-0">
            <p class="text-sm font-medium text-gray-900 dark:text-gray-100 truncate">
              {{ item.name }}
            </p>
            <p class="text-xs text-gray-400 dark:text-gray-500">
              {{ formatSize(item.size) }}<span v-if="item.pageCount !== undefined"> · {{ item.pageCount }} page{{ item.pageCount === 1 ? '' : 's' }}</span><span
                v-else-if="item.type === 'pdf'"
                class="italic"
              > · counting…</span>
            </p>
          </div>

          <UButton
            variant="ghost"
            size="xs"
            color="neutral"
            icon="i-lucide-x"
            class="flex-shrink-0"
            @click.stop="removeFile(item.id)"
          />
        </div>
      </div>

      <p class="text-xs text-gray-400 dark:text-gray-500 mt-2 text-center">
        Drag items to reorder
      </p>
    </div>

    <!-- Error -->
    <UAlert
      v-if="mergeError"
      color="error"
      icon="i-lucide-circle-alert"
      class="mb-4"
      :description="mergeError"
    />

    <!-- Action -->
    <UButton
      size="lg"
      class="w-full justify-center"
      :disabled="!canMerge"
      :loading="isMerging"
      icon="i-lucide-download"
      @click="merge"
    >
      {{ mergeLabel }}
    </UButton>

    <p
      v-if="files.length === 0"
      class="text-xs text-center text-gray-400 dark:text-gray-500 mt-3"
    >
      Your files never leave your browser — all processing is done locally
    </p>
  </UContainer>
</template>
