<script setup lang="ts">
import { computed, nextTick, onMounted, ref, watch } from 'vue'

type Role = 'user' | 'assistant'

type Message = {
  role: Role
  content: string
  isError?: boolean
  isCode?: boolean
  code?: string
  preview?: string
}

type Conversation = {
  id: string
  title: string
  messages: Message[]
}

const ICONS = {
  sun: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round"><circle cx="12" cy="12" r="4.2"/><path d="M12 2.5v2.4M12 19.1v2.4M4.9 4.9l1.7 1.7M17.4 17.4l1.7 1.7M2.5 12h2.4M19.1 12h2.4M4.9 19.1l1.7-1.7M17.4 6.6l1.7-1.7"/></svg>`,
  moon: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M20 14.5A8.5 8.5 0 1 1 9.5 4a6.8 6.8 0 0 0 10.5 10.5Z"/></svg>`,
  plus: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.1" stroke-linecap="round"><path d="M12 5v14M5 12h14"/></svg>`,
  trash: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.9" stroke-linecap="round" stroke-linejoin="round"><path d="M4 7h16M9 7V5a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v2m2 0-.8 12.1a2 2 0 0 1-2 1.9H9.8a2 2 0 0 1-2-1.9L7 7"/></svg>`,
  send: `<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><path d="M4 12h15M13 6l6 6-6 6"/></svg>`,
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

const uid = () => Math.random().toString(36).slice(2, 10)

const makeConversation = (title = 'Percakapan baru'): Conversation => ({
  id: uid(),
  title,
  messages: [],
})

const theme = ref<'dark' | 'light'>('dark')
const conversations = ref<Conversation[]>([makeConversation()])
const activeId = ref<string>(conversations.value[0]?.id ?? '')
const input = ref('')
const isSending = ref(false)
const confirmingDelete = ref<string | null>(null)
const sidebarOpen = ref(false)
const messagesEl = ref<HTMLElement | null>(null)
const previewCode = ref<string | null>(null)

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

const askPuterAi = async (prompt: string) => {
  const puterClient = (window as any).puter
  if (!puterClient?.ai?.chat) return null

  try {
    const response = await Promise.race([
      puterClient.ai.chat(prompt),
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
  if (text.includes('buat') || text.includes('tulis') || text.includes('konten')) return 'Saya bisa bantu menulis ide, ringkasan, draft pesan, atau konten singkat. Coba kirim topik yang ingin Anda kembangkan.'
  if (text.includes('aplikasi') || text.includes('website') || text.includes('project')) return 'Untuk aplikasi atau website, biasanya langkah terbaik adalah mulai dari kebutuhan pengguna, lalu struktur fitur, UI, dan alur kerja yang sederhana.'
  return `Saya menerima pesan Anda: “${prompt}”. Saya siap bantu menjelaskan ide, membuat ringkasan, atau menyusun jawaban yang lebih rapi.`
}

const toggleTheme = () => {
  theme.value = theme.value === 'dark' ? 'light' : 'dark'
}

const newChat = () => {
  const conversation = makeConversation()
  conversations.value.unshift(conversation)
  activeId.value = conversation.id
  input.value = ''
  sidebarOpen.value = false
}

const selectChat = (id: string) => {
  activeId.value = id
  sidebarOpen.value = false
  confirmingDelete.value = null
}

const requestDelete = (id: string, event: Event) => {
  event.stopPropagation()
  if (confirmingDelete.value === id) {
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

  confirmingDelete.value = id
  window.setTimeout(() => {
    if (confirmingDelete.value === id) confirmingDelete.value = null
  }, 2200)
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
    const textarea = document.createElement('textarea')
    textarea.value = code
    document.body.appendChild(textarea)
    textarea.select()
    document.execCommand('copy')
    textarea.remove()
  }
}

const sendMessage = async () => {
  const text = input.value.trim()
  if (!text || isSending.value) return

  const conversation = active.value
  if (!conversation) return

  if (conversation.messages.length === 0) {
    conversation.title = text.length > 36 ? `${text.slice(0, 33)}...` : text
  }

  conversation.messages.push({ role: 'user', content: text })
  input.value = ''
  isSending.value = true
  await scrollToBottom()

  try {
    await new Promise((resolve) => window.setTimeout(resolve, 450))

    const puterReply = await askPuterAi(text)
    const generatedCode = generateCodeSnippet(text)

    if (puterReply) {
      conversation.messages.push({ role: 'assistant', content: String(puterReply) })
    } else if (generatedCode) {
      conversation.messages.push(generatedCode)
    } else {
      conversation.messages.push({ role: 'assistant', content: buildReply(text) })
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
  }
}

const onKeydown = (event: KeyboardEvent) => {
  if (event.key === 'Enter' && !event.shiftKey) {
    event.preventDefault()
    void sendMessage()
  }
}

const handleTextareaInput = (event: Event) => {
  const target = event.target as HTMLTextAreaElement | null
  if (!target) return
  target.style.height = 'auto'
  target.style.height = `${target.scrollHeight}px`
}

watch(
  theme,
  (value) => document.documentElement.setAttribute('data-theme', value),
  { immediate: true },
)

watch(
  () => active.value?.messages.length,
  async () => {
    await scrollToBottom()
  },
  { flush: 'post' },
)

onMounted(async () => {
  document.documentElement.setAttribute('data-theme', theme.value)
  await scrollToBottom()
})
</script>

<template>
  <div class="shell">
    <div v-if="sidebarOpen" class="overlay" @click="sidebarOpen = false" />

    <aside class="sidebar" :class="{ open: sidebarOpen }">
      <div class="brand">
        <div class="brand-mark" v-html="NODE_MARK" />
        <div class="brand-name">Nexus<span>AI</span></div>
      </div>

      <button class="new-chat-btn" @click="newChat">
        <span class="icon-wrap" v-html="ICONS.plus" />
        <span>Percakapan baru</span>
      </button>

      <div class="conversation-list">
        <button
          v-for="conversation in conversations"
          :key="conversation.id"
          class="conversation-item"
          :class="{ active: conversation.id === activeId }"
          @click="selectChat(conversation.id)"
        >
          <span class="dot" />
          <span class="title">{{ conversation.title }}</span>
          <span
            class="delete-btn"
            :class="{ confirming: confirmingDelete === conversation.id }"
            @click.stop="requestDelete(conversation.id, $event)"
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
          <button class="nav-button mobile-only" @click="sidebarOpen = !sidebarOpen" aria-label="Buka sidebar">
            <span class="icon-wrap" v-html="ICONS.plus" />
          </button>
          <div>
            <div class="page-title">{{ active?.title ?? 'NexusAI' }}</div>
            <div class="page-subtitle">NexusAI • siap membantu</div>
          </div>
        </div>

        <button class="theme-toggle" @click="toggleTheme" :title="theme === 'dark' ? 'Mode terang' : 'Mode gelap'">
          <span class="icon-wrap" v-html="theme === 'dark' ? ICONS.sun : ICONS.moon" />
        </button>
      </header>

      <section class="messages" ref="messagesEl">
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
            <div v-if="message.role === 'assistant'" class="avatar" v-html="NODE_MARK" />

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
                  <iframe :srcdoc="message.preview ?? ''" title="Preview code" class="preview-frame" />
                </div>
              </div>
            </template>

            <div v-else class="bubble" :class="{ error: message.isError }">{{ message.content }}</div>

            <div v-if="message.role === 'user'" class="user-badge">U</div>
          </div>

          <div v-if="isSending" class="message-row assistant">
            <div class="avatar" v-html="NODE_MARK" />
            <div class="bubble typing"><span /><span /><span /></div>
          </div>
        </div>
      </section>

      <footer class="composer-wrap">
        <div class="composer">
          <textarea
            rows="1"
            v-model="input"
            placeholder="Kirim pesan ke NexusAI..."
            @keydown="onKeydown"
            @input="handleTextareaInput"
          />
          <button class="send-btn" :disabled="!input.trim() || isSending" @click="sendMessage" aria-label="Kirim pesan">
            <span class="icon-wrap" v-html="ICONS.send" />
          </button>
        </div>
        <div class="composer-hint">Enter untuk kirim • Shift + Enter untuk baris baru</div>
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

.shell {
  display: grid;
  grid-template-columns: 280px 1fr;
  height: 100%;
  background: var(--bg);
}

.sidebar {
  background: var(--surface);
  border-right: 1px solid var(--border);
  display: flex;
  flex-direction: column;
  min-width: 0;
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
}

.main-panel {
  display: flex;
  flex-direction: column;
  min-width: 0;
  min-height: 0;
  height: 100%;
}

.topbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 22px;
  border-bottom: 1px solid var(--border);
}

.topbar-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.page-title {
  font-family: 'Sora', sans-serif;
  font-size: 15px;
  font-weight: 700;
}

.page-subtitle {
  font-size: 11.5px;
  color: var(--text-faint);
  margin-top: 2px;
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
}

.message-row.user {
  justify-content: flex-end;
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
  max-width: min(82%, 620px);
  padding: 12px 14px;
  border-radius: var(--radius);
  background: var(--surface-2);
  border: 1px solid var(--border);
  color: var(--text);
  line-height: 1.6;
  white-space: pre-wrap;
  word-break: break-word;
}

.message-row.user .bubble {
  background: linear-gradient(135deg, var(--accent), #8b5cf6);
  color: #fff;
  border-color: transparent;
}

.bubble.error {
  background: var(--danger-soft);
  border-color: rgba(242, 85, 90, 0.5);
  color: var(--danger);
}

.code-card {
  width: min(760px, 100%);
  background: var(--surface-2);
  border: 1px solid var(--border);
  border-radius: 18px;
  padding: 12px;
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
  padding: 10px 14px;
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
  padding: 16px 22px 24px;
}

.composer {
  max-width: 760px;
  margin: 0 auto;
  display: flex;
  align-items: flex-end;
  gap: 12px;
  padding: 8px 8px 8px 16px;
  border: 1px solid var(--border);
  border-radius: 18px;
  background: var(--surface-2);
}

.composer:focus-within {
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(132, 103, 255, 0.12);
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

  .message-list {
    padding: 0 16px;
  }

  .bubble {
    max-width: 88%;
  }
}
</style>
