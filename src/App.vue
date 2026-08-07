<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref } from 'vue'
import { compressToBase64, decompressFromBase64 } from 'lz-string'

const fileInput = ref<HTMLInputElement | null>(null)
const editor = ref<HTMLTextAreaElement | null>(null)
const highlightedCode = ref<HTMLElement | null>(null)
const lineNumbers = ref<HTMLElement | null>(null)
const findInput = ref<HTMLInputElement | null>(null)
const fileName = ref('')
const fileSize = ref(0)
const jsonText = ref('')
const originalText = ref('')
const errorMessage = ref('')
const dragActive = ref(false)
const cursorLine = ref(1)
const cursorColumn = ref(1)
const notice = ref('')
const isFindOpen = ref(false)
const searchQuery = ref('')
const activeMatchIndex = ref(0)
let noticeTimer: ReturnType<typeof setTimeout> | undefined

const hasFile = computed(() => Boolean(fileName.value))
const isDirty = computed(() => hasFile.value && jsonText.value !== originalText.value)
const lineCount = computed(() => Math.max(1, jsonText.value.split('\n').length))
const parsedJson = computed(() => {
  if (!hasFile.value) return { valid: false, value: null as unknown }
  try {
    return { valid: true, value: JSON.parse(jsonText.value) as unknown }
  } catch {
    return { valid: false, value: null as unknown }
  }
})
const formattedSize = computed(() =>
  fileSize.value < 1024 ? `${fileSize.value} B` : `${(fileSize.value / 1024).toFixed(1)} KB`,
)
const searchMatches = computed(() => {
  if (!searchQuery.value) return []
  const text = jsonText.value.toLocaleLowerCase()
  const query = searchQuery.value.toLocaleLowerCase()
  const matches: number[] = []
  let offset = 0
  while (offset <= text.length - query.length) {
    const index = text.indexOf(query, offset)
    if (index === -1) break
    matches.push(index)
    offset = index + Math.max(query.length, 1)
  }
  return matches
})
const currentMatchNumber = computed(() =>
  searchMatches.value.length ? Math.min(activeMatchIndex.value + 1, searchMatches.value.length) : 0,
)

function escapeHtml(value: string) {
  return value.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
}

function renderTextSegment(segment: string, absoluteStart: number) {
  if (!searchQuery.value || !searchMatches.value.length) return escapeHtml(segment)
  const segmentEnd = absoluteStart + segment.length
  const activeStart = searchMatches.value[activeMatchIndex.value]
  let cursor = 0
  let html = ''

  for (const matchStart of searchMatches.value) {
    const matchEnd = matchStart + searchQuery.value.length
    if (matchEnd <= absoluteStart) continue
    if (matchStart >= segmentEnd) break
    const localStart = Math.max(matchStart, absoluteStart) - absoluteStart
    const localEnd = Math.min(matchEnd, segmentEnd) - absoluteStart
    html += escapeHtml(segment.slice(cursor, localStart))
    const currentClass = matchStart === activeStart ? ' search-current' : ''
    html += `<mark class="search-match${currentClass}">${escapeHtml(segment.slice(localStart, localEnd))}</mark>`
    cursor = localEnd
  }
  return html + escapeHtml(segment.slice(cursor))
}

const highlightedJson = computed(() => {
  const tokenPattern = /("(?:\\u[\da-fA-F]{4}|\\[^u]|[^\\"])*"\s*:)|("(?:\\u[\da-fA-F]{4}|\\[^u]|[^\\"])*")|\b(true|false)\b|\b(null)\b|(-?\d+(?:\.\d+)?(?:[eE][+-]?\d+)?)/g
  let html = ''
  let lastIndex = 0

  for (const match of jsonText.value.matchAll(tokenPattern)) {
    const index = match.index ?? 0
    html += renderTextSegment(jsonText.value.slice(lastIndex, index), lastIndex)
    const name = match[1]
      ? 'token-key'
      : match[2]
        ? 'token-string'
        : match[3]
          ? 'token-boolean'
          : match[4]
            ? 'token-null'
            : 'token-number'
    html += `<span class="${name}">${renderTextSegment(match[0], index)}</span>`
    lastIndex = index + match[0].length
  }
  return html + renderTextSegment(jsonText.value.slice(lastIndex), lastIndex)
})

function showNotice(message: string) {
  notice.value = message
  window.clearTimeout(noticeTimer)
  noticeTimer = window.setTimeout(() => (notice.value = ''), 2400)
}

function openFilePicker() {
  fileInput.value?.click()
}

async function handleFile(file?: File) {
  if (!file) return
  errorMessage.value = ''
  if (!file.name.toLowerCase().endsWith('.rpgsave')) {
    errorMessage.value = '请选择 .rpgsave 文件'
    return
  }
  try {
    const encoded = (await file.text()).replace(/^\uFEFF/, '').trim()
    const decompressed = decompressFromBase64(encoded)
    if (!decompressed) throw new Error('无法解码文件内容')
    const parsed = JSON.parse(decompressed) as unknown
    const formatted = JSON.stringify(parsed, null, 2)
    fileName.value = file.name
    fileSize.value = file.size
    jsonText.value = formatted
    originalText.value = formatted
    cursorLine.value = 1
    cursorColumn.value = 1
    searchQuery.value = ''
    activeMatchIndex.value = 0
    isFindOpen.value = false
    showNotice('存档已成功解码')
    await nextTick()
    if (editor.value) {
      editor.value.scrollTop = 0
      editor.value.scrollLeft = 0
      editor.value.focus()
      syncScroll()
    }
  } catch (error) {
    errorMessage.value = error instanceof Error ? error.message : '文件解析失败'
  } finally {
    if (fileInput.value) fileInput.value.value = ''
  }
}

function handleInputChange(event: Event) {
  void handleFile((event.target as HTMLInputElement).files?.[0])
}
function handleDrop(event: DragEvent) {
  dragActive.value = false
  void handleFile(event.dataTransfer?.files[0])
}
function resetContent() {
  jsonText.value = originalText.value
  errorMessage.value = ''
  showNotice('已恢复为导入时的内容')
}
function downloadSave() {
  if (!hasFile.value || !parsedJson.value.valid) return
  const encoded = compressToBase64(JSON.stringify(parsedJson.value.value))
  const url = URL.createObjectURL(new Blob([encoded], { type: 'application/octet-stream' }))
  const link = document.createElement('a')
  link.href = url
  link.download = `${fileName.value.replace(/\.rpgsave$/i, '')}_modified.rpgsave`
  document.body.appendChild(link)
  link.click()
  link.remove()
  URL.revokeObjectURL(url)
  showNotice('修改后的存档已下载')
}
function syncScroll() {
  if (!editor.value) return
  if (highlightedCode.value) {
    highlightedCode.value.style.transform = `translate(${-editor.value.scrollLeft}px, ${-editor.value.scrollTop}px)`
  }
  if (lineNumbers.value) lineNumbers.value.style.transform = `translateY(${-editor.value.scrollTop}px)`
}
function updateCursor() {
  if (!editor.value) return
  const lines = jsonText.value.slice(0, editor.value.selectionStart).split('\n')
  cursorLine.value = lines.length
  cursorColumn.value = (lines.at(-1)?.length ?? 0) + 1
}

function openFind() {
  if (!hasFile.value) return
  isFindOpen.value = true
  nextTick(() => {
    findInput.value?.focus()
    findInput.value?.select()
  })
}

function closeFind() {
  isFindOpen.value = false
  nextTick(() => editor.value?.focus())
}

function jumpToMatch(index: number) {
  const matches = searchMatches.value
  if (!matches.length || !editor.value) return
  activeMatchIndex.value = (index + matches.length) % matches.length
  const start = matches[activeMatchIndex.value]!
  const end = start + searchQuery.value.length
  const beforeMatch = jsonText.value.slice(0, start).split('\n')
  const line = beforeMatch.length
  const column = beforeMatch.at(-1)?.length ?? 0

  editor.value.focus()
  editor.value.setSelectionRange(start, end)
  editor.value.scrollTop = Math.max(0, (line - 4) * 24)
  editor.value.scrollLeft = Math.max(0, (column - 12) * 7.8)
  updateCursor()
  syncScroll()
  nextTick(() => findInput.value?.focus())
}

function refreshFind() {
  activeMatchIndex.value = 0
  nextTick(() => {
    if (searchMatches.value.length) jumpToMatch(0)
  })
}

function findNext(direction = 1) {
  if (!searchMatches.value.length) return
  jumpToMatch(activeMatchIndex.value + direction)
}

function handleWindowKeydown(event: KeyboardEvent) {
  if ((event.ctrlKey || event.metaKey) && event.key.toLowerCase() === 'f' && hasFile.value) {
    event.preventDefault()
    openFind()
  } else if (event.key === 'Escape' && isFindOpen.value) {
    event.preventDefault()
    closeFind()
  }
}

function handleEditorKeydown(event: KeyboardEvent) {
  if ((event.ctrlKey || event.metaKey) && event.key.toLowerCase() === 's') {
    event.preventDefault()
    downloadSave()
    return
  }
  if (event.key !== 'Tab' || !editor.value) return
  event.preventDefault()
  const start = editor.value.selectionStart
  const end = editor.value.selectionEnd
  jsonText.value = `${jsonText.value.slice(0, start)}  ${jsonText.value.slice(end)}`
  nextTick(() => {
    if (!editor.value) return
    editor.value.selectionStart = editor.value.selectionEnd = start + 2
    updateCursor()
  })
}

onMounted(() => window.addEventListener('keydown', handleWindowKeydown))
onBeforeUnmount(() => {
  window.removeEventListener('keydown', handleWindowKeydown)
  window.clearTimeout(noticeTimer)
})
</script>

<template>
  <main
    class="app-shell"
    @dragenter.prevent="dragActive = true"
    @dragover.prevent="dragActive = true"
    @dragleave.self.prevent="dragActive = false"
    @drop.prevent="handleDrop"
  >
    <header class="topbar">
      <div class="brand" aria-label="RPG 存档修改器">
        <span class="brand-mark" aria-hidden="true"></span>
        <span>RPG Save Editor</span>
      </div>
      <div class="header-actions">
        <button v-if="hasFile" class="button button-ghost" type="button" @click="openFilePicker">
          <span class="icon icon-upload" aria-hidden="true"></span> 更换文件
        </button>
        <button
          class="button button-primary"
          type="button"
          :disabled="!hasFile || !parsedJson.valid"
          @click="downloadSave"
        >
          <span class="icon icon-download" aria-hidden="true"></span> 下载存档
        </button>
      </div>
    </header>

    <section class="workspace" :class="{ 'workspace-empty': !hasFile }">
      <template v-if="!hasFile">
        <div class="empty-state" @click="openFilePicker">
          <div class="upload-icon" aria-hidden="true"></div>
          <h1>打开 RPG Maker 存档</h1>
          <p>拖放文件到这里，或点击选择</p>
          <button class="button button-primary choose-button" type="button">选择 .rpgsave 文件</button>
          <p class="privacy-note"><span aria-hidden="true">◆</span> 文件仅在本地浏览器中处理</p>
        </div>
        <p v-if="errorMessage" class="empty-error" role="alert">{{ errorMessage }}</p>
      </template>

      <template v-else>
        <div class="filebar">
          <div class="file-info">
            <span class="file-icon" aria-hidden="true"></span>
            <div>
              <strong>{{ fileName }}</strong>
              <span>{{ formattedSize }} · LZString / Base64</span>
            </div>
          </div>
          <div class="file-actions">
            <span v-if="isDirty" class="changed-indicator"><i></i> 已修改</span>
            <button class="text-button" type="button" :disabled="!isDirty" @click="resetContent">恢复原文</button>
          </div>
        </div>

        <div class="editor-frame">
          <div class="editor-labelbar">
            <span>JSON</span>
            <div v-if="isFindOpen" class="findbar" role="search">
              <input
                ref="findInput"
                v-model="searchQuery"
                type="text"
                aria-label="在 JSON 中查找"
                placeholder="查找"
                autocomplete="off"
                @input="refreshFind"
                @keydown.enter.prevent="findNext($event.shiftKey ? -1 : 1)"
                @keydown.esc.prevent="closeFind"
              />
              <span class="find-count">{{ currentMatchNumber }}/{{ searchMatches.length }}</span>
              <button type="button" title="上一个匹配" aria-label="上一个匹配" @mousedown.prevent @click="findNext(-1)">↑</button>
              <button type="button" title="下一个匹配" aria-label="下一个匹配" @mousedown.prevent @click="findNext(1)">↓</button>
              <button class="find-close" type="button" title="关闭查找" aria-label="关闭查找" @click="closeFind">×</button>
            </div>
            <span v-else class="validity" :class="parsedJson.valid ? 'is-valid' : 'is-invalid'">
              <i aria-hidden="true"></i>{{ parsedJson.valid ? '格式有效' : 'JSON 格式错误' }}
            </span>
          </div>
          <div class="editor-wrap">
            <div class="gutter" aria-hidden="true">
              <div ref="lineNumbers" class="line-numbers">
                <span v-for="line in lineCount" :key="line">{{ line }}</span>
              </div>
            </div>
            <pre ref="highlightedCode" class="highlight-layer" aria-hidden="true"><code v-html="highlightedJson"></code></pre>
            <textarea
              ref="editor"
              v-model="jsonText"
              class="code-input"
              aria-label="JSON 存档内容编辑器"
              wrap="off"
              spellcheck="false"
              autocapitalize="off"
              autocomplete="off"
              @scroll="syncScroll"
              @click="updateCursor"
              @keyup="updateCursor"
              @select="updateCursor"
              @keydown="handleEditorKeydown"
            ></textarea>
          </div>
          <div class="statusbar">
            <span>行 {{ cursorLine }}，列 {{ cursorColumn }}</span><span>UTF-8</span><span>空格：2</span>
          </div>
        </div>
      </template>
    </section>

    <input ref="fileInput" class="visually-hidden" type="file" accept=".rpgsave" @change="handleInputChange" />
    <div v-if="dragActive" class="drop-overlay" aria-hidden="true"><div><span class="icon icon-upload drop-icon"></span>释放以打开存档</div></div>
    <Transition name="toast"><div v-if="notice" class="toast" role="status">✓ {{ notice }}</div></Transition>
  </main>
</template>

<style>
:root {
  color: #172033;
  background: #f4f7fb;
  font-family: Inter, "SF Pro Text", "PingFang SC", "Microsoft YaHei", system-ui, sans-serif;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
}
* { box-sizing: border-box; }
html, body, #app { min-width: 320px; min-height: 100%; margin: 0; }
body { min-height: 100vh; }
button, textarea { font: inherit; }
button { border: 0; }
.app-shell { min-height: 100vh; background: #f4f7fb; }
.topbar {
  height: 66px; padding: 0 clamp(20px, 4vw, 52px); display: flex; align-items: center;
  justify-content: space-between; background: #fff; border-bottom: 1px solid #e7ebf2;
}
.brand { display: flex; align-items: center; gap: 11px; font-size: 15px; font-weight: 700; }
.brand-mark, .file-icon { display: grid; place-items: center; color: #fff; background: #2878f0; }
.brand-mark { width: 30px; height: 30px; border-radius: 8px; box-shadow: 0 5px 14px #2878f038; }
.brand-mark::before {
  content: ''; width: 19px; height: 19px; background: #fff;
  -webkit-mask: url('/code.svg') center / contain no-repeat;
  mask: url('/code.svg') center / contain no-repeat;
}
.file-icon::before {
  content: ''; width: 20px; height: 20px; background: #fff;
  -webkit-mask: url('/file.svg') center / contain no-repeat;
  mask: url('/file.svg') center / contain no-repeat;
}
.icon { display: inline-block; width: 14px; height: 14px; flex: 0 0 14px; background: currentColor; }
.icon-upload {
  -webkit-mask: url('/upload.svg') center / contain no-repeat;
  mask: url('/upload.svg') center / contain no-repeat;
}
.icon-download {
  -webkit-mask: url('/download.svg') center / contain no-repeat;
  mask: url('/download.svg') center / contain no-repeat;
}
.header-actions, .file-actions { display: flex; align-items: center; gap: 10px; }
.button {
  min-height: 38px; padding: 0 16px; display: inline-flex; align-items: center; justify-content: center;
  gap: 8px; border-radius: 8px; font-size: 13px; font-weight: 650; cursor: pointer; transition: 160ms ease;
}
.button:focus-visible, .text-button:focus-visible, .code-input:focus-visible { outline: 2px solid #2878f06b; outline-offset: 2px; }
.button-primary { color: #fff; background: #2878f0; box-shadow: 0 4px 11px #2878f033; }
.button-primary:hover:not(:disabled) { background: #1f68d6; transform: translateY(-1px); }
.button:disabled { color: #9ca6b8; background: #e8edf4; box-shadow: none; cursor: not-allowed; }
.button-ghost { color: #344157; background: #fff; border: 1px solid #dfe5ee; }
.button-ghost:hover { color: #2878f0; border-color: #b8d2fa; background: #f7faff; }
.workspace {
  width: min(1480px, calc(100% - clamp(28px, 6vw, 80px))); height: calc(100vh - 98px); min-height: 560px;
  margin: 16px auto; display: flex; flex-direction: column; overflow: hidden; background: #fff;
  border: 1px solid #e4e9f1; border-radius: 13px; box-shadow: 0 8px 28px #1f31500d;
}
.workspace-empty { align-items: center; justify-content: center; }
.empty-state {
  width: min(520px, calc(100% - 40px)); min-height: 332px; padding: 46px 30px 34px; display: flex;
  flex-direction: column; align-items: center; border: 1.5px dashed #cbd9ed; border-radius: 14px;
  background: #fbfdff; cursor: pointer; transition: 180ms ease;
}
.empty-state:hover { border-color: #6ca4f6; background: #f8fbff; transform: translateY(-2px); box-shadow: 0 12px 30px #2878f012; }
.upload-icon {
  width: 62px; height: 62px; display: grid; place-items: center; margin-bottom: 22px; color: #2878f0;
  background: #eaf3ff; border-radius: 16px;
}
.upload-icon::before {
  content: ''; width: 28px; height: 28px; background: currentColor;
  -webkit-mask: url('/upload.svg') center / contain no-repeat;
  mask: url('/upload.svg') center / contain no-repeat;
}
.empty-state h1 { margin: 0 0 9px; color: #18243a; font-size: 20px; line-height: 1.35; letter-spacing: -.025em; }
.empty-state p { margin: 0; color: #7e899b; font-size: 13px; }
.choose-button { margin: 23px 0 20px; pointer-events: none; }
.empty-state .privacy-note { color: #98a2b2; font-size: 11px; }
.privacy-note span { margin-right: 5px; color: #5d9af4; font-size: 8px; }
.empty-error { margin: 17px 0 0; color: #d14646; font-size: 13px; }
.filebar {
  height: 72px; flex: 0 0 72px; padding: 0 20px; display: flex; align-items: center;
  justify-content: space-between; border-bottom: 1px solid #ebeff5;
}
.file-info { min-width: 0; display: flex; align-items: center; gap: 12px; }
.file-icon { width: 36px; height: 36px; flex: 0 0 36px; border-radius: 9px; font-size: 12px; }
.file-info div { min-width: 0; display: flex; flex-direction: column; gap: 4px; }
.file-info strong { max-width: min(50vw, 560px); overflow: hidden; font-size: 13px; font-weight: 650; text-overflow: ellipsis; white-space: nowrap; }
.file-info div span { color: #929cad; font-size: 11px; }
.changed-indicator, .validity { display: inline-flex; align-items: center; gap: 6px; color: #697487; font-size: 11px; }
.changed-indicator i, .validity i { width: 6px; height: 6px; border-radius: 50%; background: #2878f0; }
.text-button { padding: 7px 8px; color: #2878f0; background: transparent; border-radius: 6px; font-size: 12px; cursor: pointer; }
.text-button:hover:not(:disabled) { background: #eff6ff; }
.text-button:disabled { color: #b0b8c6; cursor: default; }
.editor-frame {
  min-height: 0; margin: 14px; flex: 1; display: flex; flex-direction: column; overflow: hidden;
  border: 1px solid #e1e6ee; border-radius: 9px;
}
.editor-labelbar {
  height: 38px; flex: 0 0 38px; padding: 0 14px; display: flex; align-items: center; justify-content: space-between;
  color: #687489; background: #fafbfd; border-bottom: 1px solid #e7ebf1; font-size: 10px; font-weight: 700; letter-spacing: .08em;
}
.findbar { display: flex; align-items: center; gap: 3px; height: 30px; letter-spacing: 0; }
.findbar input {
  width: min(230px, 28vw); height: 28px; padding: 0 58px 0 9px; color: #26344d; background: #fff;
  border: 1px solid #ccd6e4; border-radius: 6px; outline: none; font-size: 12px; font-weight: 400;
}
.findbar input:focus { border-color: #6ba3f3; box-shadow: 0 0 0 2px #2878f01f; }
.find-count { position: relative; width: 0; right: 52px; color: #909aac; font-size: 10px; font-weight: 400; white-space: nowrap; }
.findbar button {
  width: 27px; height: 27px; padding: 0; display: grid; place-items: center; color: #627087;
  background: transparent; border-radius: 5px; font-size: 14px; cursor: pointer;
}
.findbar button:hover { color: #216edc; background: #eaf3ff; }
.findbar .find-close { margin-left: 2px; font-size: 18px; font-weight: 300; }
.validity { letter-spacing: 0; font-weight: 500; }
.validity.is-valid { color: #36835b; } .validity.is-valid i { background: #44a873; }
.validity.is-invalid { color: #c94c4c; } .validity.is-invalid i { background: #e05252; }
.editor-wrap { position: relative; min-height: 0; flex: 1; overflow: hidden; background: #fff; }
.highlight-layer, .code-input {
  position: absolute; inset: 0; width: 100%; height: 100%; margin: 0; padding: 18px 24px 24px 72px; border: 0;
  font: 400 13px/24px "SFMono-Regular", Consolas, "Liberation Mono", Menlo, monospace;
  font-variant-ligatures: none; letter-spacing: 0; word-spacing: 0; tab-size: 2; white-space: pre;
}
.highlight-layer { width: max-content; min-width: 100%; height: max-content; min-height: 100%; overflow: visible; color: #35415a; pointer-events: none; }
.highlight-layer code { font: inherit; letter-spacing: inherit; word-spacing: inherit; }
.code-input { z-index: 2; display: block; resize: none; color: transparent; caret-color: #226edb; background: transparent; -webkit-text-fill-color: transparent; overflow: auto; }
.code-input::selection { background: #2878f030; } .code-input:focus { outline: none; }
.gutter {
  position: absolute; z-index: 3; inset: 0 auto 0 0; width: 55px; overflow: hidden; color: #b1bac8;
  background: #fbfcfe; border-right: 1px solid #edf0f5; pointer-events: none;
}
.line-numbers { padding-top: 18px; display: flex; flex-direction: column; font-family: Consolas, monospace; font-size: 11px; line-height: 24px; text-align: right; }
.line-numbers span { height: 24px; padding-right: 13px; }
.token-key { color: #1565c0; } .token-string { color: #14805e; } .token-number { color: #8a55c5; }
.token-boolean { color: #d26517; } .token-null { color: #9b5b4a; font-style: italic; }
.search-match { padding: 0; color: inherit; background: #ffe69a; border-radius: 2px; }
.search-match.search-current { color: #1f2b3f; background: #ffb64d; box-shadow: 0 0 0 1px #e6921f; }
.statusbar {
  height: 30px; flex: 0 0 30px; padding: 0 13px; display: flex; align-items: center; justify-content: flex-end;
  gap: 18px; color: #9aa3b2; background: #fafbfd; border-top: 1px solid #e7ebf1; font-family: Consolas, monospace; font-size: 10px;
}
.drop-overlay {
  position: fixed; z-index: 20; inset: 12px; display: grid; place-items: center; color: #1763c8;
  background: #eff7fff0; border: 2px dashed #5b9af5; border-radius: 16px; backdrop-filter: blur(3px); pointer-events: none;
}
.drop-overlay div { display: flex; align-items: center; gap: 12px; font-size: 17px; font-weight: 700; }
.drop-overlay .drop-icon { width: 25px; height: 25px; flex-basis: 25px; }
.toast { position: fixed; z-index: 30; right: 24px; bottom: 24px; padding: 11px 16px; color: #fff; background: #23324a; border-radius: 8px; box-shadow: 0 10px 30px #121d2f33; font-size: 12px; }
.toast-enter-active, .toast-leave-active { transition: 180ms ease; }
.toast-enter-from, .toast-leave-to { opacity: 0; transform: translateY(8px); }
.visually-hidden { position: fixed; width: 1px; height: 1px; overflow: hidden; clip: rect(0, 0, 0, 0); clip-path: inset(50%); white-space: nowrap; }
@media (max-width: 680px) {
  .topbar { height: 60px; padding: 0 14px; } .brand span:last-child { display: none; } .button { padding: 0 12px; }
  .workspace { width: calc(100% - 16px); height: calc(100vh - 76px); min-height: 500px; margin: 8px auto; border-radius: 10px; }
  .filebar { height: 64px; flex-basis: 64px; padding: 0 12px; } .file-info strong { max-width: 36vw; }
  .changed-indicator { display: none; } .editor-frame { margin: 8px; }
  .highlight-layer, .code-input { padding-left: 58px; font-size: 12px; } .gutter { width: 44px; }
  .line-numbers span { padding-right: 9px; } .statusbar { gap: 12px; } .statusbar span:nth-child(2) { display: none; }
  .findbar input { width: min(160px, 42vw); }
}
@media (prefers-reduced-motion: reduce) { *, *::before, *::after { transition-duration: .01ms !important; } }
</style>
