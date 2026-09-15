<script setup lang="ts">
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'

type Role = 'user' | 'assistant'

type Attachment = {
  name: string
  size: number
  type: string
  kind: 'image' | 'file'
  url?: string
  dataUrl?: string
}

type Message = {
  role: Role
  content: string
  isError?: boolean
  isCode?: boolean
  code?: string
  preview?: string
  attachments?: Attachment[]
}

type Conversation = {
  id: string
  title: string
  messages: Message[]
}

type Theme = 'dark' | 'light'

const ICONS = {
  sun: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><circle cx="12" cy="12" r="4.2"/><path d="M12 2.5v2.4M12 19.1v2.4M4.9 4.9l1.7 1.7M17.4 17.4l1.7 1.7M2.5 12h2.4M19.1 12h2.4M4.9 19.1l1.7-1.7M17.4 6.6l1.7-1.7"/></svg>`,
  moon: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M20 14.5A8.5 8.5 0 1 1 9.5 4a6.8 6.8 0 0 0 10.5 10.5Z"/></svg>`,
  plus: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.1" stroke-linecap="round"><path d="M12 5v14M5 12h14"/></svg>`,
  trash: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="M4 7h16M9 7V5a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v2m2 0-.8 12.1a2 2 0 0 1-2 1.9H9.8a2 2 0 0 1-2-1.9L7 7"/></svg>`,
  send: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 12h15M13 6l6 6-6 6"/></svg>`,
  menu: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><path d="M4 6h16M4 12h16M4 18h16"/></svg>`,
} as const

const NODE_MARK = `
  <svg viewBox="0 0 32 32" fill="none" xmlns="http://www.w3.org/2000/svg" style="display:block;width:100%;height:100%;color:var(--accent)">
    <g stroke="currentColor" stroke-width="1.8" stroke-linecap="round" stroke-linejoin="round">
      <line x1="8" y1="8" x2="24" y2="10"/>
      <line x1="8" y1="8" x2="12" y2="24"/>
      <line x1="24" y1="10" x2="12" y2="24"/>
      <line x1="24" y1="10" x2="26" y2="22" opacity="0.8"/>
    </g>
    <circle cx="8" cy="8" r="3.8" fill="currentColor"/>
    <circle cx="24" cy="10" r="3.2" fill="currentColor"/>
    <circle cx="12" cy="24" r="3.8" fill="currentColor"/>
    <circle cx="26" cy="22" r="2.4" fill="currentColor" opacity="0.75"/>
  </svg>`

const STORAGE_KEYS = {
  conversations: 'nexusai.conversations.v1',
  activeId: 'nexusai.activeId.v1',
  theme: 'nexusai.theme.v1',
} as const

const uid = () => Math.random().toString(36).slice(2, 10)

const makeConversation = (title = 'Percakapan baru'): Conversation => ({
  id: uid(),
  title,
  messages: [],
})

// --- Safe localStorage helpers (private browsing / disabled storage / SSR won't throw) ---
const safeStorage = {
  get(key: string): string | null {
    try {
      return typeof window === 'undefined' ? null : window.localStorage.getItem(key)
    } catch {
      return null
    }
  },
  set(key: string, value: string) {
    try {
      if (typeof window !== 'undefined') window.localStorage.setItem(key, value)
    } catch {
      /* storage unavailable — fail silently, app still works in-memory */
    }
  },
}

const loadInitialConversations = (): Conversation[] => {
  const raw = safeStorage.get(STORAGE_KEYS.conversations)
  if (!raw) return [makeConversation()]
  try {
    const parsed = JSON.parse(raw)
    if (Array.isArray(parsed) && parsed.length > 0) {
      // Basic shape validation so corrupted/old data can't crash the app.
      const valid = parsed.every(
        (c) => c && typeof c.id === 'string' && typeof c.title === 'string' && Array.isArray(c.messages),
      )
      if (valid) return parsed
    }
  } catch {
    /* fall through to default */
  }
  return [makeConversation()]
}

const loadInitialTheme = (): Theme => {
  const raw = safeStorage.get(STORAGE_KEYS.theme)
  return raw === 'light' || raw === 'dark' ? raw : 'dark'
}

const theme = ref<Theme>(loadInitialTheme())
const conversations = ref<Conversation[]>(loadInitialConversations())
const initialActiveId = safeStorage.get(STORAGE_KEYS.activeId)
const activeId = ref<string>(
  (initialActiveId && conversations.value.some((c) => c.id === initialActiveId)
    ? initialActiveId
    : conversations.value[0]?.id) ?? '',
)
const input = ref('')
const uploadedFiles = ref<Attachment[]>([])
const isSending = ref(false)
const confirmingDelete = ref<string | null>(null)
const sidebarOpen = ref(false)
const messagesEl = ref<HTMLElement | null>(null)
const textareaEl = ref<HTMLTextAreaElement | null>(null)
const fileInputEl = ref<HTMLInputElement | null>(null)
const previewCode = ref<string | null>(null)
let deleteConfirmTimeout: number | null = null

const active = computed<Conversation | null>(
  () => conversations.value.find((conversation) => conversation.id === activeId.value) ?? conversations.value[0] ?? null,
)

const extractText = (value: unknown): string | null => {
  if (typeof value === 'string') return value
  if (Array.isArray(value)) {
    const joined = value
      .map((item) => (typeof item === 'string' ? item : typeof item === 'object' && item && 'text' in item ? String((item as { text?: unknown }).text ?? '') : ''))
      .join('\n')
    return joined || null
  }
  if (value && typeof value === 'object') {
    if ('content' in value) {
      const content = (value as { content?: unknown }).content
      const text = extractText(content)
      if (text) return text
    }
    if ('message' in value) {
      const message = (value as { message?: unknown }).message
      const text = extractText(message)
      if (text) return text
    }
  }
  return null
}

const readFileAsDataUrl = (file: File): Promise<string> =>
  new Promise((resolve, reject) => {
    const reader = new FileReader()
    reader.onload = () => resolve(String(reader.result ?? ''))
    reader.onerror = () => reject(new Error('Failed to read file'))
    reader.readAsDataURL(file)
  })

const askPuterAi = async (prompt: string, attachments: Attachment[] = []) => {
  const puterClient = (window as any).puter
  if (!puterClient?.ai?.chat) return null

  try {
    const imageAttachment = attachments.find((attachment) => attachment.kind === 'image' && attachment.dataUrl)

    const response = await Promise.race([
      imageAttachment?.dataUrl
        ? puterClient.ai.chat(prompt, imageAttachment.dataUrl, { model: 'gpt-5.6-luna' })
        : puterClient.ai.chat(prompt),
      new Promise((_, reject) => window.setTimeout(() => reject(new Error('Puter AI timeout')), 15000)),
    ])

    const text = extractText(response)
    return text ?? null
  } catch (error) {
    console.error('Puter AI error:', error)
    return null
  }
}

const generateCodeSnippet = (prompt: string): Message | null => {
  const text = prompt.trim().toLowerCase()
  const wantsCode = /kode|html|css|javascript|js|component|button|card/i.test(text)
  if (!wantsCode) return null

  const label = /button/i.test(text) ? 'button' : /card/i.test(text) ? 'card' : 'landing page'
  const code = `<!DOCTYPE html>
<html lang="id">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>${label}</title>
    <style>
      * { box-sizing: border-box; }
      body {
        margin: 0;
        min-height: 100vh;
        display: grid;
        place-items: center;
        background: linear-gradient(135deg, #0f172a, #1e293b);
        font-family: Arial, sans-serif;
        color: white;
      }
      .card {
        width: min(420px, 90vw);
        padding: 28px;
        border-radius: 18px;
        background: rgba(15, 23, 42, 0.8);
        border: 1px solid rgba(148, 163, 184, 0.3);
        box-shadow: 0 20px 50px rgba(15, 23, 42, 0.5);
      }
      h1 {
        margin: 0 0 12px;
        font-size: 2rem;
      }
      p {
        margin: 0 0 18px;
        color: #cbd5e1;
        line-height: 1.6;
      }
      button {
        border: none;
        border-radius: 999px;
        padding: 12px 20px;
        background: linear-gradient(135deg, #8b5cf6, #22d3ee);
        color: white;
        font-weight: 700;
        cursor: pointer;
      }
    </style>
  </head>
  <body>
    <div class="card">
      <h1>${label.toUpperCase()}</h1>
      <p>Ini contoh kode yang bisa langsung Anda copy dan sesuaikan.</p>
      <button>Mulai sekarang</button>
    </div>
  </body>
</html>`

  return {
    role: 'assistant',
    content: 'Berikut contoh kode yang bisa langsung Anda copy.',
    isCode: true,
    code,
    preview: code,
  }
}

const buildReply = (prompt: string) => {
  const text = prompt.trim().toLowerCase()
  if (!text) return 'Saya belum menerima pertanyaan apa pun.'
  if (text.includes('halo') || text.includes('hai') || text.includes('hello')) return 'Halo! Saya NexusAI. Ada yang bisa saya bantu hari ini?'
  if (text.includes('siapa') && text.includes('kamu')) return 'Saya adalah NexusAI, asisten yang bisa bantu menulis, menjelaskan ide, dan membuat draft.'
  if (
    (text.includes('siapa') || text.includes('nama')) &&
    (text.includes('pembuat') || text.includes('creator') || text.includes('developer') || text.includes('buat') || text.includes('membuat')) &&
    (text.includes('kamu') || text.includes('nexus') || text.includes('ai'))
  ) {
    return 'Pembuat saya adalah Adun.'
  }
  if (text.includes('buat') || text.includes('tulis') || text.includes('konten')) return 'Saya bisa bantu menulis ide, ringkasan, draft pesan, atau konten singkat. Coba kirim topik yang ingin Anda kembangkan.'
  if (text.includes('aplikasi') || text.includes('website') || text.includes('project')) return 'Untuk aplikasi atau website, biasanya langkah terbaik adalah mulai dari kebutuhan pengguna, lalu struktur fitur, UI, dan alur kerja yang sederhana.'
  return `Saya menerima pesan Anda: "${prompt}". Saya siap bantu menjelaskan ide, membuat ringkasan, atau menyusun jawaban yang lebih rapi.`
}

const toggleTheme = () => {
  theme.value = theme.value === 'dark' ? 'light' : 'dark'
}

const resetTextareaHeight = () => {
  if (textareaEl.value) textareaEl.value.style.height = 'auto'
}

const newChat = () => {
  const conversation = makeConversation()
  conversations.value.unshift(conversation)
  activeId.value = conversation.id
  input.value = ''
  sidebarOpen.value = false
  resetTextareaHeight()
}

const selectChat = (id: string) => {
  activeId.value = id
  sidebarOpen.value = false
  confirmingDelete.value = null
}

const clearDeleteTimeout = () => {
  if (deleteConfirmTimeout !== null) {
    window.clearTimeout(deleteConfirmTimeout)
    deleteConfirmTimeout = null
  }
}

const requestDelete = (id: string, event: Event) => {
  event.stopPropagation()

  if (confirmingDelete.value === id) {
    clearDeleteTimeout()
    const index = conversations.value.findIndex((conversation) => conversation.id === id)
    if (index !== -1) conversations.value.splice(index, 1)
    if (conversations.value.length === 0) {
      const conversation = makeConversation()
      conversations.value.push(conversation)
      activeId.value = conversation.id
    } else if (activeId.value === id) {
      activeId.value = conversations.value[0]!.id
    }
    confirmingDelete.value = null
    return
  }

  clearDeleteTimeout()
  confirmingDelete.value = id
  deleteConfirmTimeout = window.setTimeout(() => {
    if (confirmingDelete.value === id) confirmingDelete.value = null
    deleteConfirmTimeout = null
  }, 2200)
}

const escapeHtml = (value: string) =>
  value
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#039;')

const formatMessageContent = (value: string) => {
  const segments = value.split(/```([\s\S]*?)```/g)

  return segments
    .map((segment, index) => {
      if (index % 2 === 1) {
        return `<pre class="formatted-code"><code>${escapeHtml(segment.trim())}</code></pre>`
      }

      const safeText = escapeHtml(segment)
      return safeText
        .replace(/\*\*(.+?)\*\*/g, '<strong>$1</strong>')
        .replace(/\n/g, '<br>')
    })
    .join('')
}

const formatFileSize = (bytes: number) => {
  if (bytes < 1024) return `${bytes} B`
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`
}

const createAttachmentFromFile = async (file: File): Promise<Attachment> => {
  const imageKind = file.type.startsWith('image/')
  const dataUrl = imageKind ? await readFileAsDataUrl(file) : undefined

  return {
    name: file.name,
    size: file.size,
    type: file.type,
    kind: imageKind ? 'image' : 'file',
    ...(imageKind ? { url: URL.createObjectURL(file), dataUrl } : {}),
  }
}

const triggerFilePicker = () => {
  fileInputEl.value?.click()
}

const clearUploadedFiles = () => {
  uploadedFiles.value.forEach((file) => {
    if (file.url) URL.revokeObjectURL(file.url)
  })
  uploadedFiles.value = []
}

const handleFileSelection = async (event: Event) => {
  const target = event.target as HTMLInputElement | null
  const files = Array.from(target?.files ?? [])

  if (!files.length || !target) return

  const attachments = await Promise.all(files.map((file) => createAttachmentFromFile(file)))
  uploadedFiles.value = [...uploadedFiles.value, ...attachments]
  target.value = ''
}

const removeAttachment = (index: number) => {
  const [removed] = uploadedFiles.value.splice(index, 1)
  if (removed?.url) URL.revokeObjectURL(removed.url)
}

const scrollToBottom = async () => {
  await nextTick()
  requestAnimationFrame(() => {
    if (messagesEl.value) {
      messagesEl.value.scrollTop = messagesEl.value.scrollHeight
    }
  })
}

const copyCode = async (code: string) => {
  try {
    await navigator.clipboard.writeText(code)
  } catch {
    try {
      const textarea = document.createElement('textarea')
      textarea.value = code
      textarea.style.position = 'fixed'
      textarea.style.opacity = '0'
      document.body.appendChild(textarea)
      textarea.select()
      document.execCommand('copy')
      textarea.remove()
    } catch {
      /* clipboard unavailable — nothing more we can safely do */
    }
  }
}

const sendMessage = async () => {
  const text = input.value.trim()
  const hasAttachments = uploadedFiles.value.length > 0

  if ((!text && !hasAttachments) || isSending.value) return

  const conversation = active.value
  if (!conversation) return

  const attachmentNames = uploadedFiles.value.map((file) => file.name)
  const finalPrompt = text || `Mengirim ${attachmentNames.length} file: ${attachmentNames.join(', ')}`

  if (conversation.messages.length === 0) {
    const titleSource = text || attachmentNames[0] || 'Percakapan baru'
    conversation.title = titleSource.length > 36 ? `${titleSource.slice(0, 33)}...` : titleSource
  }

  const sentAttachments = uploadedFiles.value.slice()
  const singleAttachmentName = sentAttachments[0]?.name ?? 'file'
  conversation.messages.push({
    role: 'user',
    content: text || (sentAttachments.length === 1 ? `Mengirim file: ${singleAttachmentName}` : `Mengirim ${sentAttachments.length} file`),
    attachments: sentAttachments.length ? sentAttachments : undefined,
  })

  uploadedFiles.value = []
  input.value = ''
  resetTextareaHeight()
  isSending.value = true
  await scrollToBottom()

  try {
    await new Promise((resolve) => window.setTimeout(resolve, 450))

    const puterReply = await askPuterAi(finalPrompt, sentAttachments)
    const generatedCode = generateCodeSnippet(finalPrompt)

    if (puterReply) {
      conversation.messages.push({ role: 'assistant', content: String(puterReply) })
    } else if (generatedCode) {
      conversation.messages.push(generatedCode)
    } else {
      const fallbackText = hasAttachments
        ? `Saya menerima file berikut: ${attachmentNames.join(', ')}. Jika Anda mau, saya bisa meninjau isinya, meringkasnya, atau membantu mengubahnya menjadi format lain.`
        : buildReply(finalPrompt)
      conversation.messages.push({ role: 'assistant', content: fallbackText })
    }
  } catch {
    conversation.messages.push({
      role: 'assistant',
      content: 'Gagal menghubungi NexusAI. Silakan coba lagi sebentar lagi.',
      isError: true,
    })
  } finally {
    isSending.value = false
    await scrollToBottom()
    textareaEl.value?.focus()
  }
}

const onKeydown = (event: KeyboardEvent) => {
  if (event.key === 'Enter' && !event.shiftKey) {
    event.preventDefault()
    void sendMessage()
  }
}

const handlePaste = async (event: ClipboardEvent) => {
  const items = Array.from(event.clipboardData?.items ?? [])
  const imageFiles = items
    .filter((item) => item.kind === 'file' && item.type.startsWith('image/'))
    .map((item) => item.getAsFile())
    .filter((file): file is File => Boolean(file))

  if (!imageFiles.length) return

  event.preventDefault()
  const attachments = await Promise.all(imageFiles.map((file) => createAttachmentFromFile(file)))
  uploadedFiles.value = [...uploadedFiles.value, ...attachments]
}

const handleTextareaInput = (event: Event) => {
  const target = event.target as HTMLTextAreaElement | null
  if (!target) return
  target.style.height = 'auto'
  target.style.height = `${target.scrollHeight}px`
}

const onGlobalKeydown = (event: KeyboardEvent) => {
  if (event.key === 'Escape' && sidebarOpen.value) {
    sidebarOpen.value = false
  }
}

watch(
  theme,
  (value) => {
    document.documentElement.setAttribute('data-theme', value)
    safeStorage.set(STORAGE_KEYS.theme, value)
  },
  { immediate: true },
)

watch(
  conversations,
  (value) => {
    safeStorage.set(STORAGE_KEYS.conversations, JSON.stringify(value))
  },
  { deep: true },
)

watch(activeId, (value) => {
  safeStorage.set(STORAGE_KEYS.activeId, value)
})

watch(
  () => active.value?.messages.length,
  async () => {
    await scrollToBottom()
  },
  { flush: 'post' },
)

onMounted(async () => {
  document.documentElement.setAttribute('data-theme', theme.value)
  window.addEventListener('keydown', onGlobalKeydown)
  await scrollToBottom()
})

onBeforeUnmount(() => {
  window.removeEventListener('keydown', onGlobalKeydown)
  clearDeleteTimeout()
  clearUploadedFiles()
})
</script>

<template>
  <div class="shell">
    <div v-if="sidebarOpen" class="overlay" @click="sidebarOpen = false" />

    <aside class="sidebar" :class="{ open: sidebarOpen }" aria-label="Daftar percakapan">
      <div class="brand">
        <div class="brand-mark" v-html="NODE_MARK" />
        <div class="brand-name">Nexus<span>AI</span></div>
      </div>

      <button class="new-chat-btn" @click="newChat">
        <span class="icon-wrap" v-html="ICONS.plus" />
        <span>Percakapan baru</span>
      </button>

      <div class="conversation-list" role="list">
        <button
          v-for="conversation in conversations"
          :key="conversation.id"
          class="conversation-item"
          :class="{ active: conversation.id === activeId }"
          role="listitem"
          :aria-current="conversation.id === activeId ? 'true' : undefined"
          @click="selectChat(conversation.id)"
        >
          <span class="dot" />
          <span class="title">{{ conversation.title }}</span>
          <span
            class="delete-btn"
            :class="{ confirming: confirmingDelete === conversation.id }"
            role="button"
            tabindex="0"
            :aria-label="confirmingDelete === conversation.id ? 'Konfirmasi hapus percakapan' : 'Hapus percakapan'"
            @click.stop="requestDelete(conversation.id, $event)"
            @keydown.enter.stop="requestDelete(conversation.id, $event)"
            @keydown.space.stop.prevent="requestDelete(conversation.id, $event)"
            :title="confirmingDelete === conversation.id ? 'Hapus sekarang' : 'Hapus percakapan'"
          >
            <span v-if="confirmingDelete === conversation.id">Yakin?</span>
            <span v-else class="icon-wrap" v-html="ICONS.trash" />
          </span>
        </button>
      </div>

      <div class="sidebar-footer">{{ conversations.length }} percakapan tersimpan</div>
    </aside>

    <main class="main-panel">
      <header class="topbar">
        <div class="topbar-left">
          <button class="nav-button mobile-only" @click="sidebarOpen = !sidebarOpen" aria-label="Buka daftar percakapan">
            <span class="icon-wrap" v-html="ICONS.menu" />
          </button>
          <div class="topbar-titles">
            <div class="page-title">{{ active?.title ?? 'NexusAI' }}</div>
            <div class="page-subtitle">NexusAI • siap membantu</div>
          </div>
        </div>

        <button
          class="theme-toggle"
          @click="toggleTheme"
          :aria-label="theme === 'dark' ? 'Aktifkan mode terang' : 'Aktifkan mode gelap'"
          :title="theme === 'dark' ? 'Mode terang' : 'Mode gelap'"
        >
          <span class="icon-wrap" v-html="theme === 'dark' ? ICONS.sun : ICONS.moon" />
        </button>
      </header>

      <section class="messages" ref="messagesEl" role="log" aria-live="polite" aria-label="Riwayat percakapan">
        <div v-if="!active || active.messages.length === 0" class="empty-state">
          <div class="empty-mark" v-html="NODE_MARK" />
          <div class="empty-title">Mulai percakapan dengan NexusAI</div>
          <div class="empty-sub">Tanyakan apa saja — ide, penjelasan, bantuan menulis, atau sekadar mengobrol.</div>
        </div>

        <div v-else class="message-list">
          <div
            v-for="(message, index) in active.messages"
            :key="`${message.role}-${index}`"
            class="message-row"
            :class="message.role"
          >
            <div v-if="message.role === 'assistant'" class="avatar" aria-hidden="true" v-html="NODE_MARK" />

            <template v-if="message.isCode">
              <div class="code-card">
                <div class="code-header">
                  <span>Kode siap copy</span>
                  <div class="code-actions">
                    <button class="mini-btn" @click="copyCode(message.code ?? '')">Copy</button>
                    <button class="mini-btn" @click="previewCode = previewCode === message.preview ? null : message.preview ?? null">
                      {{ previewCode === message.preview ? 'Tutup preview' : 'Preview' }}
                    </button>
                  </div>
                </div>
                <pre class="code-block">{{ message.code }}</pre>
                <div v-if="previewCode === message.preview" class="preview-panel">
                  <iframe
                    :srcdoc="message.preview ?? ''"
                    title="Preview kode"
                    class="preview-frame"
                    sandbox="allow-scripts allow-modals"
                    referrerpolicy="no-referrer"
                    loading="lazy"
                  />
                </div>
              </div>
            </template>

            <div
              v-else
              class="bubble"
              :class="{ error: message.isError }"
              v-html="formatMessageContent(message.content)"
            />

            <div v-if="message.role === 'user' && message.attachments?.length" class="attachment-list">
              <div v-for="(attachment, attachmentIndex) in message.attachments" :key="`${attachment.name}-${attachmentIndex}`" class="attachment-item">
                <img
                  v-if="attachment.kind === 'image' && attachment.url"
                  :src="attachment.url"
                  class="attachment-image"
                  :alt="attachment.name"
                />
                <div class="attachment-meta" :class="{ file: attachment.kind !== 'image' }">
                  <span class="attachment-name">{{ attachment.name }}</span>
                  <span class="attachment-size">{{ formatFileSize(attachment.size) }}</span>
                </div>
              </div>
            </div>

            <div v-if="message.role === 'user'" class="user-badge" aria-hidden="true">U</div>
          </div>

          <div v-if="isSending" class="message-row assistant" aria-hidden="true">
            <div class="avatar" v-html="NODE_MARK" />
            <div class="bubble typing"><span /><span /><span /></div>
          </div>
        </div>
      </section>

      <footer class="composer-wrap">
        <div v-if="uploadedFiles.length" class="upload-preview">
          <div v-for="(file, index) in uploadedFiles" :key="`${file.name}-${index}`" class="upload-chip">
            <span class="upload-name">{{ file.name }}</span>
            <button type="button" class="upload-remove" @click="removeAttachment(index)" aria-label="Hapus file">×</button>
          </div>
        </div>

        <div class="composer">
          <button type="button" class="upload-btn" @click="triggerFilePicker" aria-label="Upload file">
            <span class="icon-wrap">📎</span>
          </button>
          <input ref="fileInputEl" type="file" class="sr-only" multiple @change="handleFileSelection" />

          <label for="composer-input" class="sr-only">Tulis pesan</label>
          <textarea
            id="composer-input"
            ref="textareaEl"
            rows="1"
            v-model="input"
            placeholder="Kirim pesan ke NexusAI..."
            :disabled="isSending"
            @keydown="onKeydown"
            @paste="handlePaste"
            @input="handleTextareaInput"
          />
          <button class="send-btn" :disabled="(!input.trim() && !uploadedFiles.length) || isSending" @click="sendMessage" aria-label="Kirim pesan">
            <span class="icon-wrap" v-html="ICONS.send" />
          </button>
        </div>
        <div class="composer-hint">Enter untuk kirim • Shift + Enter untuk baris baru • lampiran file didukung</div>
      </footer>
    </main>
  </div>
</template>

<style scoped>
:root {
  --radius: 16px;
  --radius-sm: 10px;
  --radius-pill: 999px;
}

:global(html) {
  background: var(--bg);
}

:global(body) {
  margin: 0;
  min-height: 100vh;
  background: var(--bg);
  color: var(--text);
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  -webkit-font-smoothing: antialiased;
}

:global(html[data-theme='dark']) {
  --bg: #0b0d12;
  --surface: #12151c;
  --surface-2: #1a1e27;
  --surface-3: #20242f;
  --surface-4: #0f172a;
  --border: #262b36;
  --text: #e7e9ee;
  --text-dim: #8b92a5;
  --text-faint: #5f697d;
  --accent: #8467ff;
  --accent-soft: rgba(132, 103, 255, 0.14);
  --signal: #37d9c4;
  --danger: #f2555a;
  --danger-soft: rgba(242, 85, 90, 0.14);
  --shadow: 0 18px 40px rgba(0, 0, 0, 0.35);
}

:global(html[data-theme='light']) {
  --bg: #f5f5fa;
  --surface: #ffffff;
  --surface-2: #f1f2f7;
  --surface-3: #e7e9f2;
  --surface-4: #eef2ff;
  --border: #e2e5ef;
  --text: #14151c;
  --text-dim: #6d7284;
  --text-faint: #9197a6;
  --accent: #6f5af8;
  --accent-soft: rgba(111, 90, 248, 0.09);
  --signal: #12a796;
  --danger: #dd4148;
  --danger-soft: rgba(221, 65, 72, 0.09);
  --shadow: 0 14px 28px rgba(20, 21, 30, 0.08);
}

:global(*) {
  box-sizing: border-box;
}

:global(#app) {
  height: 100vh;
}

button,
textarea {
  font: inherit;
}

button {
  touch-action: manipulation;
}

.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}

:deep(:focus-visible) {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
  border-radius: 4px;
}

.shell {
  display: grid;
  grid-template-columns: 280px 1fr;
  height: 100%;
  height: 100dvh;
  background: var(--bg);
}

.sidebar {
  background: var(--surface);
  border-right: 1px solid var(--border);
  display: flex;
  flex-direction: column;
  min-width: 0;
  min-height: 0;
}

.brand {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 22px 18px 16px;
}

.brand-mark {
  width: 30px;
  height: 30px;
  display: block;
  color: var(--accent);
  flex-shrink: 0;
}

.brand-mark svg {
  display: block;
  width: 100%;
  height: 100%;
}

.brand-name {
  font-family: 'Sora', sans-serif;
  font-size: 17px;
  font-weight: 700;
  letter-spacing: -0.02em;
}

.brand-name span {
  color: var(--accent);
}

.new-chat-btn {
  margin: 4px 14px 14px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 10px 12px;
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  background: var(--surface-2);
  color: var(--text);
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, border-color 0.15s ease;
}

.new-chat-btn:hover {
  background: var(--surface-3);
  border-color: var(--accent);
}

.conversation-list {
  flex: 1;
  padding: 0 10px 10px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  overflow-y: auto;
  min-height: 0;
}

.conversation-item {
  border: 1px solid transparent;
  background: transparent;
  color: var(--text-dim);
  border-radius: var(--radius-sm);
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 10px;
  cursor: pointer;
  text-align: left;
  transition: background 0.15s ease, color 0.15s ease;
}

.conversation-item:hover {
  background: var(--surface-2);
  color: var(--text);
}

.conversation-item.active {
  background: var(--accent-soft);
  color: var(--text);
  border-color: rgba(132, 103, 255, 0.28);
}

.dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--text-faint);
  flex-shrink: 0;
}

.conversation-item.active .dot {
  background: var(--accent);
}

.title {
  flex: 1;
  min-width: 0;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 13.5px;
}

.delete-btn {
  flex-shrink: 0;
  min-width: 24px;
  height: 24px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--text-faint);
  background: transparent;
  font-size: 10px;
  font-weight: 700;
  transition: background 0.15s ease, color 0.15s ease;
}

.delete-btn:hover {
  background: var(--danger-soft);
  color: var(--danger);
}

.delete-btn.confirming {
  background: var(--danger);
  color: white;
  width: auto;
  padding: 0 8px;
}

.sidebar-footer {
  border-top: 1px solid var(--border);
  padding: 12px 18px 16px;
  color: var(--text-faint);
  font-size: 11px;
  flex-shrink: 0;
}

.main-panel {
  display: flex;
  flex-direction: column;
  min-width: 0;
  min-height: 0;
  height: 100%;
  overflow: hidden;
}

.topbar {
  position: sticky;
  top: 0;
  z-index: 10;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 22px;
  border-bottom: 1px solid var(--border);
  background: color-mix(in srgb, var(--surface) 90%, transparent);
  backdrop-filter: blur(10px);
  flex-shrink: 0;
}

.topbar-left {
  display: flex;
  align-items: center;
  gap: 12px;
  min-width: 0;
}

.topbar-titles {
  min-width: 0;
}

.page-title {
  font-family: 'Sora', sans-serif;
  font-size: 15px;
  font-weight: 700;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.page-subtitle {
  font-size: 11.5px;
  color: var(--text-faint);
  margin-top: 2px;
  white-space: nowrap;
}

.nav-button,
.theme-toggle {
  width: 38px;
  height: 38px;
  border: 1px solid var(--border);
  border-radius: var(--radius-pill);
  background: var(--surface-2);
  color: var(--text);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  flex-shrink: 0;
  transition: background 0.15s ease;
}

.nav-button:hover,
.theme-toggle:hover {
  background: var(--surface-3);
}

.icon-wrap {
  width: 16px;
  height: 16px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  color: currentColor;
  pointer-events: none;
}

.icon-wrap svg {
  width: 100%;
  height: 100%;
  display: block;
}

.messages {
  flex: 1;
  min-height: 0;
  overflow-y: auto;
  overflow-x: hidden;
  padding: 26px 0;
  scroll-behavior: smooth;
  overscroll-behavior: contain;
}

.empty-state {
  max-width: 520px;
  margin: 14vh auto 0;
  text-align: center;
  padding: 0 24px;
}

.empty-mark {
  width: 52px;
  height: 52px;
  margin: 0 auto 18px;
  color: var(--accent);
}

.empty-mark svg {
  width: 100%;
  height: 100%;
  display: block;
}

.empty-title {
  font-family: 'Sora', sans-serif;
  font-size: 20px;
  font-weight: 700;
  margin-bottom: 8px;
}

.empty-sub {
  color: var(--text-dim);
  line-height: 1.6;
  font-size: 14px;
}

.message-list {
  max-width: 760px;
  margin: 0 auto;
  padding: 0 24px;
  display: flex;
  flex-direction: column;
  gap: 18px;
}

.message-row {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  width: 100%;
  min-width: 0;
  max-width: 100%;
}

.message-row > * {
  min-width: 0;
}

.message-row.user {
  justify-content: flex-end;
  min-width: 0;
}

.avatar,
.user-badge {
  width: 28px;
  height: 28px;
  border-radius: 50%;
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}

.avatar {
  color: var(--accent);
  background: var(--surface-2);
  border: 1px solid var(--border);
}

.avatar svg {
  width: 14px;
  height: 14px;
  display: block;
}

.user-badge {
  background: linear-gradient(135deg, #3b82f6, #8b5cf6);
  color: white;
  font-weight: 700;
  font-size: 12px;
}

.bubble {
  display: block;
  width: auto;
  max-width: min(82%, 620px);
  min-width: 0;
  padding: 12px 18px;
  border-radius: 22px;
  background: var(--surface-2);
  border: 1px solid var(--border);
  color: var(--text);
  line-height: 1.6;
  white-space: normal;
  overflow-wrap: anywhere;
  word-break: break-word;
  box-sizing: border-box;
  box-shadow: 0 6px 18px rgba(15, 23, 42, 0.08);
}

.bubble strong {
  font-weight: 800;
}

.bubble .formatted-code {
  margin: 10px 0 0;
  padding: 12px 14px;
  border-radius: 12px;
  background: #0b1120;
  border: 1px solid rgba(148, 163, 184, 0.25);
  overflow-x: auto;
  white-space: pre-wrap;
  overflow-wrap: anywhere;
  word-break: break-word;
  color: #dbeafe;
  user-select: text;
}

.bubble .formatted-code code {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 12.5px;
  line-height: 1.6;
  white-space: pre-wrap;
}

.message-row.user .bubble {
  background: linear-gradient(135deg, var(--accent), #8b5cf6);
  color: #fff;
  border-color: transparent;
  border-radius: 22px;
}

.bubble.error {
  background: var(--danger-soft);
  border-color: rgba(242, 85, 90, 0.5);
  color: var(--danger);
}

.attachment-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-top: 12px;
  width: min(100%, 420px);
}

.attachment-item {
  display: flex;
  align-items: center;
  gap: 10px;
  background: rgba(148, 163, 184, 0.08);
  border: 1px solid var(--border);
  border-radius: 14px;
  padding: 8px 10px;
}

.attachment-image {
  width: 56px;
  height: 56px;
  object-fit: cover;
  border-radius: 10px;
  border: 1px solid var(--border);
  flex-shrink: 0;
  background: var(--surface-3);
}

.attachment-meta {
  display: flex;
  flex-direction: column;
  min-width: 0;
  gap: 3px;
}

.attachment-meta.file {
  padding-left: 2px;
}

.attachment-name {
  font-size: 12.5px;
  font-weight: 600;
  color: var(--text);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.attachment-size {
  font-size: 11px;
  color: var(--text-dim);
}

.code-card {
  width: min(100%, 760px);
  max-width: 100%;
  background: var(--surface-2);
  border: 1px solid var(--border);
  border-radius: 22px;
  padding: 12px;
  box-sizing: border-box;
  box-shadow: var(--shadow);
}

.code-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
  margin-bottom: 10px;
  color: var(--text-dim);
  font-size: 12px;
  font-weight: 700;
}

.code-actions {
  display: flex;
  gap: 8px;
}

.mini-btn {
  border: 1px solid var(--border);
  background: var(--surface-3);
  color: var(--text);
  border-radius: 10px;
  padding: 7px 10px;
  cursor: pointer;
}

.mini-btn:hover {
  border-color: var(--accent);
}

.code-block {
  margin: 0;
  padding: 14px;
  border-radius: 12px;
  background: #0b1120;
  color: #dbeafe;
  border: 1px solid rgba(148, 163, 184, 0.25);
  white-space: pre-wrap;
  word-break: break-word;
  font-size: 12.5px;
  line-height: 1.6;
  overflow-x: auto;
}

.preview-panel {
  margin-top: 12px;
  border-radius: 12px;
  overflow: hidden;
  border: 1px solid var(--border);
}

.preview-frame {
  display: block;
  width: 100%;
  min-height: 280px;
  border: none;
  background: white;
}

.typing {
  display: flex;
  align-items: center;
  gap: 4px;
  padding: 10px 18px;
  border-radius: 22px;
}

.typing span {
  width: 6px;
  height: 6px;
  background: var(--text-faint);
  border-radius: 50%;
  animation: bounce 1.1s infinite ease-in-out;
}

.typing span:nth-child(2) { animation-delay: 0.15s; }
.typing span:nth-child(3) { animation-delay: 0.3s; }

@keyframes bounce {
  0%, 60%, 100% { transform: translateY(0); opacity: 0.5; }
  30% { transform: translateY(-4px); opacity: 1; }
}

.composer-wrap {
  border-top: 1px solid var(--border);
  padding: 16px 22px calc(env(safe-area-inset-bottom, 0px) + 24px);
  flex-shrink: 0;
}

.composer {
  max-width: 760px;
  margin: 0 auto;
  display: flex;
  align-items: flex-end;
  gap: 12px;
  padding: 8px 8px 8px 12px;
  border: 1px solid var(--border);
  border-radius: 18px;
  background: var(--surface-2);
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
}

.composer:focus-within {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(132, 103, 255, 0.12);
}

.upload-btn {
  width: 38px;
  height: 38px;
  border: 1px solid var(--border);
  border-radius: 12px;
  background: var(--surface-3);
  color: var(--text);
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  flex-shrink: 0;
}

.upload-btn:hover {
  border-color: var(--accent);
}

.upload-preview {
  max-width: 760px;
  margin: 0 auto 10px;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.upload-chip {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 7px 10px;
  background: var(--surface-2);
  border: 1px solid var(--border);
  border-radius: 999px;
  color: var(--text);
}

.upload-name {
  max-width: 180px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  font-size: 12px;
}

.upload-remove {
  border: none;
  background: transparent;
  color: var(--text-faint);
  cursor: pointer;
  font-size: 18px;
  line-height: 1;
  padding: 0;
}

.composer textarea {
  flex: 1;
  border: none;
  outline: none;
  resize: none;
  background: transparent;
  color: var(--text);
  font-size: 14.5px;
  line-height: 1.6;
  max-height: 180px;
  min-height: 44px;
  padding: 8px 0;
}

.composer textarea:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.composer textarea::placeholder {
  color: var(--text-faint);
}

.send-btn {
  width: 42px;
  height: 42px;
  border: none;
  border-radius: 12px;
  background: var(--accent);
  color: white;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
  transition: filter 0.15s ease;
}

.send-btn:disabled {
  cursor: default;
  opacity: 0.4;
}

.send-btn:hover:not(:disabled) {
  filter: brightness(1.04);
}

.composer-hint {
  text-align: center;
  margin-top: 8px;
  font-size: 11px;
  color: var(--text-faint);
}

.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.35);
  z-index: 15;
}

.mobile-only {
  display: none;
}

@media (max-width: 760px) {
  .shell {
    grid-template-columns: 1fr;
  }

  .sidebar {
    position: fixed;
    inset: 0 auto 0 0;
    width: 260px;
    z-index: 20;
    transform: translateX(-105%);
    transition: transform 0.2s ease;
    box-shadow: var(--shadow);
  }

  .sidebar.open {
    transform: translateX(0);
  }

  .mobile-only {
    display: flex;
  }

  .topbar {
    padding: 12px 14px;
  }

  .page-title {
    max-width: 160px;
  }

  .page-subtitle {
    display: none;
  }

  .message-list {
    padding: 0 16px;
  }

  .bubble {
    max-width: 88%;
  }
}

@media (prefers-reduced-motion: reduce) {
  .sidebar,
  .new-chat-btn,
  .conversation-item,
  .delete-btn,
  .nav-button,
  .theme-toggle,
  .mini-btn,
  .send-btn,
  .composer {
    transition: none !important;
  }

  .typing span {
    animation: none !important;
    opacity: 0.8;
  }

  .messages {
    scroll-behavior: auto;
  }
}
</style>