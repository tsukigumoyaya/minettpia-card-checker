<script setup>
import { ref, computed, watch, onMounted, nextTick } from 'vue'
import Cropper from 'cropperjs'
import 'cropperjs/dist/cropper.css'

const BASE = import.meta.env.BASE_URL

const step = ref('upload') // upload | edit
const isDragging = ref(false)
const imageSrc = ref('')

const frames = ref([])
const selectedFrame = ref(null)
const serifText = ref('')
const fontSize = ref(50)
const copySuccess = ref(false)

const CANVAS_W = 1200
const CANVAS_H = 1600

const cropImageEl = ref(null)
const cropperInstance = ref(null)
const cropBoxData = ref(null)
const frameImage = ref(null)
const previewCanvas = ref(null)
const downloadWithFrame = ref(false)

const isSerifEnabled = computed(() => {
  if (!selectedFrame.value) return true
  return selectedFrame.value.id === 'frame_sr'
})

const frameImageSrc = computed(() => {
  if (!selectedFrame.value) return null
  return `${BASE}frames/${selectedFrame.value.file}`
})

const overlayFontSize = computed(() => {
  if (!cropBoxData.value) return 0
  return (cropBoxData.value.width / CANVAS_W) * fontSize.value
})

onMounted(async () => {
  try {
    const res = await fetch(`${BASE}frames.json`)
    if (!res.ok) throw new Error(res.statusText)
    frames.value = await res.json()
  } catch {
    console.error('フレーム一覧の読み込みに失敗しました')
  }
})

// --- Upload ---
function onFileChange(e) {
  const file = e.target.files?.[0]
  if (file) handleFile(file)
}

function onDrop(e) {
  isDragging.value = false
  const file = e.dataTransfer?.files?.[0]
  if (file) handleFile(file)
}

function handleFile(file) {
  if (file.size > 20 * 1024 * 1024) {
    alert('ファイルサイズは20MB以下にしてください。')
    return
  }
  if (!['image/jpeg', 'image/png', 'image/webp'].includes(file.type)) {
    alert('JPEG / PNG / WebP のみ対応しています。')
    return
  }
  const reader = new FileReader()
  reader.onload = (ev) => {
    imageSrc.value = ev.target.result
    step.value = 'edit'
    nextTick(initCropper)
  }
  reader.readAsDataURL(file)
}

// --- Cropper ---
function initCropper() {
  if (cropperInstance.value) {
    cropperInstance.value.destroy()
  }
  const img = cropImageEl.value
  if (!img) return
  cropperInstance.value = new Cropper(img, {
    aspectRatio: 3 / 4,
    viewMode: 1,
    autoCropArea: 0.9,
    responsive: true,
    background: false,
    crop() {
      updateCropBox()
    },
    ready() {
      updateCropBox()
    },
  })
}

let previewRafId = 0
function schedulePreview() {
  cancelAnimationFrame(previewRafId)
  previewRafId = requestAnimationFrame(drawPreview)
}

function updateCropBox() {
  if (!cropperInstance.value) return
  cropBoxData.value = cropperInstance.value.getCropBoxData()
  schedulePreview()
}

// --- Frame ---
let frameLoadId = 0
watch(selectedFrame, (frame) => {
  const loadId = ++frameLoadId
  if (!frame) {
    frameImage.value = null
    drawPreview()
    return
  }
  const img = new Image()
  img.crossOrigin = 'anonymous'
  img.onload = () => {
    if (loadId !== frameLoadId) return
    frameImage.value = img
    drawPreview()
  }
  img.src = `${BASE}frames/${frame.file}`
})

watch([serifText, fontSize], () => drawPreview())

// --- Preview ---
function drawPreview() {
  const canvas = previewCanvas.value
  if (!canvas || !cropperInstance.value) return
  const croppedCanvas = cropperInstance.value.getCroppedCanvas({
    width: CANVAS_W,
    height: CANVAS_H,
  })
  if (!croppedCanvas) return

  const ctx = canvas.getContext('2d')
  canvas.width = CANVAS_W
  canvas.height = CANVAS_H
  ctx.drawImage(croppedCanvas, 0, 0, CANVAS_W, CANVAS_H)

  if (frameImage.value) {
    ctx.drawImage(frameImage.value, 0, 0, CANVAS_W, CANVAS_H)
  }

  if (isSerifEnabled.value && serifText.value.trim()) {
    drawSerif(ctx)
  }
}

function drawSerif(ctx) {
  const lines = serifText.value.split('\n')
  const size = fontSize.value
  ctx.font = `${size}px sans-serif`
  ctx.fillStyle = '#000000'
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  const lineHeight = size * 1.4
  const startY = CANVAS_H * 0.83 - ((lines.length - 1) * lineHeight) / 2
  lines.forEach((line, i) => {
    ctx.fillText(line, CANVAS_W / 2, startY + i * lineHeight)
  })
}

// --- Template & Copy ---
const templateDisplayText = computed(() => {
  const serif = serifText.value.trim() || '___'
  return `@運営部
○○SR提出しました！
「${serif}」`
})

async function copyTemplate() {
  try {
    await navigator.clipboard.writeText(templateDisplayText.value)
    copySuccess.value = true
    setTimeout(() => (copySuccess.value = false), 2000)
  } catch {
    alert('コピーに失敗しました。')
  }
}

// --- Download ---
function downloadImage() {
  if (!cropperInstance.value) return
  const croppedCanvas = cropperInstance.value.getCroppedCanvas({
    width: CANVAS_W,
    height: CANVAS_H,
  })
  if (!croppedCanvas) return

  const canvas = document.createElement('canvas')
  canvas.width = CANVAS_W
  canvas.height = CANVAS_H
  const ctx = canvas.getContext('2d')

  ctx.drawImage(croppedCanvas, 0, 0, CANVAS_W, CANVAS_H)

  if (downloadWithFrame.value && frameImage.value) {
    ctx.drawImage(frameImage.value, 0, 0, CANVAS_W, CANVAS_H)
  }

  if (downloadWithFrame.value && isSerifEnabled.value && serifText.value.trim()) {
    drawSerif(ctx)
  }

  const now = new Date()
  const pad = (n) => String(n).padStart(2, '0')
  const filename = `minettepia_${pad(now.getMonth() + 1)}${pad(now.getDate())}_${pad(now.getHours())}${pad(now.getMinutes())}${pad(now.getSeconds())}.png`
  const link = document.createElement('a')
  link.download = filename
  link.href = canvas.toDataURL('image/png')
  link.click()
}

// --- Reset ---
function resetAll() {
  if (cropperInstance.value) {
    cropperInstance.value.destroy()
    cropperInstance.value = null
  }
  step.value = 'upload'
  imageSrc.value = ''
  selectedFrame.value = null
  serifText.value = ''
  fontSize.value = 50
  frameImage.value = null
  cropBoxData.value = null
}

function selectFrame(frame) {
  selectedFrame.value = selectedFrame.value?.id === frame.id ? null : frame
}

// --- Guide ---
const guideSteps = [
  { label: '写真を選択！', src: `${BASE}guide/step1.png` },
  { label: 'フレームを選択してトリミング！', src: `${BASE}guide/step2.png` },
  { label: 'プレビュー画面で確認してダウンロード！', src: `${BASE}guide/step3.png` },
]
const guideErrors = ref({})
function onGuideError(idx) {
  guideErrors.value = { ...guideErrors.value, [idx]: true }
}

</script>

<template>
  <div class="min-h-screen bg-gray-50">
    <header class="bg-gradient-to-r from-purple-400 to-purple-300 text-white py-4 px-6 shadow-lg">
      <h1 class="text-2xl font-bold">MinettePia Card Checker</h1>
    </header>

    <main class="max-w-5xl mx-auto p-4 md:p-6">
      <!-- Upload -->
      <section v-if="step === 'upload'" class="mt-8">
        <p class="text-center text-3xl text-gray-700 font-medium mb-8">
          みねとぴあのカード用に3:4へ切り抜きが出来ます！
        </p>

        <!-- Step guide -->
        <div class="grid grid-cols-3 gap-4 mb-4">
          <div v-for="(guide, i) in guideSteps" :key="i" class="text-center">
            <p class="font-semibold text-gray-700 mb-3">{{ guide.label }}</p>
            <img
              v-if="!guideErrors[i]"
              :src="guide.src"
              @error="onGuideError(i)"
              class="w-full aspect-[3/4] object-cover rounded-xl"
              :alt="guide.label"
            />
            <div
              v-else
              class="w-full aspect-[3/4] rounded-xl bg-gradient-to-br from-pink-100 to-purple-100 flex items-center justify-center border border-gray-200"
            >
              <span class="text-4xl">&#128247;</span>
            </div>
          </div>
        </div>
        <p class="text-center text-gray-500 mb-8">
          セリフ入力もしておくとテンプレート文言がコピー出来て便利かも！
        </p>

        <div
          class="border-3 border-dashed rounded-2xl p-12 text-center transition-colors"
          :class="isDragging ? 'border-pink-500 bg-pink-50' : 'border-gray-300 bg-white'"
          @dragover.prevent="isDragging = true"
          @dragleave="isDragging = false"
          @drop.prevent="onDrop"
        >
          <div class="text-5xl mb-4">&#128247;</div>
          <p class="text-lg text-gray-600 mb-4">
            写真をドラッグ＆ドロップ、またはクリックして選択
          </p>
          <p class="text-sm text-gray-400 mb-6">JPEG / PNG / WebP（最大 20MB）</p>
          <label
            class="inline-block bg-pink-500 hover:bg-pink-600 text-white font-semibold py-3 px-8 rounded-full cursor-pointer transition-colors"
          >
            ファイルを選択
            <input
              type="file"
              accept="image/jpeg,image/png,image/webp"
              class="hidden"
              @change="onFileChange"
              aria-label="写真ファイルを選択"
            />
          </label>
        </div>
      </section>

      <!-- Edit -->
      <section v-if="step === 'edit'" class="mt-8">
        <div class="flex flex-col lg:flex-row gap-6">
          <!-- Cropper with frame overlay -->
          <div class="flex-1">
            <h2 class="text-xl font-bold text-gray-800 mb-2">トリミング</h2>
            <p class="text-sm text-gray-500 mb-3">ドラッグで範囲を調整（3:4）</p>
            <div class="bg-white rounded-xl shadow p-4">
              <div class="relative">
                <img
                  ref="cropImageEl"
                  :src="imageSrc"
                  class="block max-w-full"
                  alt="トリミング対象"
                />
                <!-- Overlay on crop box -->
                <div
                  v-if="cropBoxData"
                  class="absolute pointer-events-none overflow-hidden"
                  :style="{
                    left: cropBoxData.left + 'px',
                    top: cropBoxData.top + 'px',
                    width: cropBoxData.width + 'px',
                    height: cropBoxData.height + 'px',
                    zIndex: 100,
                  }"
                >
                  <img
                    v-if="frameImageSrc"
                    :src="frameImageSrc"
                    class="absolute inset-0 w-full h-full object-fill"
                    alt=""
                  />
                  <div
                    v-if="isSerifEnabled && serifText.trim()"
                    class="absolute w-full text-center"
                    :style="{ bottom: '17%', transform: 'translateY(50%)' }"
                  >
                    <p
                      v-for="(line, i) in serifText.split('\n')"
                      :key="i"
                      class="text-black leading-snug"
                      :style="{ fontSize: overlayFontSize + 'px' }"
                    >{{ line }}</p>
                  </div>
                </div>
              </div>
            </div>
          </div>

          <!-- Right Panel -->
          <div class="lg:w-80 space-y-6">
            <!-- Frame Selection -->
            <div class="bg-white rounded-xl shadow p-4">
              <h3 class="font-bold text-gray-800 mb-3">フレーム選択</h3>
              <div class="grid grid-cols-3 gap-2">
                <button
                  v-for="frame in frames"
                  :key="frame.id"
                  class="border-2 rounded-lg p-2 text-center text-sm transition-colors cursor-pointer"
                  :class="
                    selectedFrame?.id === frame.id
                      ? 'border-pink-500 bg-pink-50'
                      : 'border-gray-200 hover:border-gray-300'
                  "
                  @click="selectFrame(frame)"
                  :aria-label="'フレーム: ' + frame.label"
                >
                  <img
                    :src="`${BASE}frames/${frame.file}`"
                    :alt="frame.label"
                    class="w-full aspect-[3/4] object-contain mb-1 rounded"
                  />
                  <span>{{ frame.label }}</span>
                </button>
              </div>
              <button
                v-if="selectedFrame"
                class="mt-2 text-sm text-gray-500 hover:text-gray-700 underline"
                @click="selectedFrame = null"
              >
                フレーム解除
              </button>
            </div>

            <!-- Serif Input -->
            <div
              class="bg-white rounded-xl shadow p-4 transition-opacity"
              :class="{ 'opacity-40 pointer-events-none': !isSerifEnabled }"
            >
              <h3 class="font-bold text-gray-800 mb-3">セリフ入力</h3>
              <p v-if="!isSerifEnabled" class="text-xs text-gray-400 mb-2">
                SRフレーム選択時のみ入力できます
              </p>
              <textarea
                v-model="serifText"
                rows="3"
                :disabled="!isSerifEnabled"
                class="w-full border border-gray-300 rounded-lg p-3 text-sm resize-none focus:outline-none focus:ring-2 focus:ring-pink-500 disabled:bg-gray-100"
                placeholder="セリフを入力..."
                aria-label="セリフテキスト入力"
              ></textarea>
              <div class="mt-3">
                <label class="text-sm text-gray-600 block mb-1">
                  フォントサイズ: {{ fontSize }}px
                </label>
                <input
                  type="range"
                  v-model.number="fontSize"
                  :disabled="!isSerifEnabled"
                  min="12"
                  max="100"
                  class="w-full accent-pink-500"
                  aria-label="フォントサイズ調整"
                />
              </div>
            </div>
          </div>
        </div>

        <!-- Preview -->
        <div class="mt-6">
          <h2 class="text-xl font-bold text-gray-800 mb-3">プレビュー</h2>
          <div class="bg-white rounded-xl shadow p-4 flex justify-center">
            <canvas
              ref="previewCanvas"
              :width="CANVAS_W"
              :height="CANVAS_H"
              class="rounded max-w-sm w-full h-auto"
            ></canvas>
          </div>
        </div>

        <!-- Template & Copy (SR only) -->
        <div v-if="isSerifEnabled" class="mt-6 bg-white rounded-xl shadow p-4">
          <h3 class="font-bold text-gray-800 mb-3">テンプレート文言</h3>
          <div class="flex items-start gap-3">
            <pre class="flex-1 bg-gray-50 rounded-lg p-3 text-sm text-gray-700 border border-gray-200 whitespace-pre-wrap font-sans">{{ templateDisplayText }}</pre>
            <button
              class="shrink-0 bg-purple-500 hover:bg-purple-600 text-white font-semibold py-2 px-5 rounded-full transition-colors"
              @click="copyTemplate"
              aria-label="テンプレートテキストをコピー"
            >
              {{ copySuccess ? 'コピー済み!' : 'コピー' }}
            </button>
          </div>
        </div>

        <!-- Actions -->
        <div class="mt-6 flex flex-wrap items-center gap-4">
          <label class="flex items-center gap-2 text-sm text-gray-700 select-none">
            <input
              type="checkbox"
              v-model="downloadWithFrame"
              class="w-4 h-4 accent-pink-500"
              aria-label="フレーム付きでダウンロード"
            />
            フレーム付きでダウンロード
          </label>
          <button
            class="bg-green-500 hover:bg-green-600 text-white font-bold py-3 px-8 rounded-full text-lg transition-colors"
            @click="downloadImage"
            aria-label="合成画像をダウンロード"
          >
            ダウンロード
          </button>
          <button
            class="bg-gray-200 hover:bg-gray-300 text-gray-700 font-semibold py-3 px-8 rounded-full transition-colors"
            @click="resetAll"
            aria-label="最初からやり直す"
          >
            やり直す
          </button>
        </div>
      </section>
    </main>
  </div>
</template>
