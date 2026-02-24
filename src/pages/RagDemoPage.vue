<template>
  <q-page class="rag-demo-page">
    <div class="page-container">
      <!-- Header -->
      <div class="page-header text-center">
        <img src="/logo_app_rag.png" alt="RAG Wiki App" width="200" />
        <div class="text-h4 text-weight-bold">RAG Demo</div>
        <div class="text-body2">Investiga temas de Wikipedia y pregunta lo que quieras</div>
      </div>

      <!-- Research Card -->
      <q-card flat bordered class="research-card">
        <q-card-section>
          <div class="text-h6 q-mb-md">
            <q-icon name="search" class="q-mr-sm" />
            Investigar tema
          </div>

          <q-input v-model.trim="topic" label="Tema" placeholder="Ej: Roman Empire, Napoleon..." outlined
            :disable="loadingResearch" @keyup.enter="onResearch">
            <template v-slot:prepend>
              <q-icon name="topic" />
            </template>
          </q-input>

          <q-btn label="Investigar" color="primary" class="full-width q-mt-md" unelevated :loading="loadingResearch"
            :disable="!topic || loadingResearch" @click="onResearch" />

          <!-- Session Info -->
          <div v-if="sessionId" class="session-info q-mt-md">
            <q-separator class="q-mb-md" />
            <div class="row q-col-gutter-sm">
              <div class="col-12">
                <div class="info-chip">
                  <q-icon name="folder" size="sm" class="q-mr-xs" />
                  <span class="text-caption">{{ sessionTopic }}</span>
                </div>
              </div>
              <div class="col-6">
                <div class="info-chip">
                  <q-icon name="description" size="sm" class="q-mr-xs" />
                  <span class="text-caption">{{ chunksIndexed }} chunks</span>
                </div>
              </div>
              <div class="col-6">
                <div class="info-chip">
                  <q-icon name="link" size="sm" class="q-mr-xs" />
                  <a :href="sourceUrl" target="_blank" class="text-caption">Ver fuente</a>
                </div>
              </div>
            </div>
          </div>
        </q-card-section>
      </q-card>

      <!-- Chat Section -->
      <q-card flat bordered class="chat-card">
        <q-card-section class="chat-header">
          <div class="row items-center">
            <div class="col">
              <div class="text-h6">
                <q-icon name="chat" class="q-mr-sm" />
                Conversación
              </div>
            </div>
            <div class="col-auto">
              <q-btn flat dense round icon="delete_outline" :disable="messages.length === 0" @click="messages = []">
                <q-tooltip>Limpiar chat</q-tooltip>
              </q-btn>
            </div>
          </div>
        </q-card-section>

        <q-separator />

        <!-- Messages -->
        <q-card-section class="chat-messages">
          <div v-if="messages.length === 0" class="empty-state">
            <q-icon name="chat_bubble_outline" size="64px" color="grey-5" />
            <div class="text-grey-7 q-mt-md">
              {{ sessionId ? 'Haz una pregunta para comenzar' : 'Primero investiga un tema' }}
            </div>
          </div>

          <div v-else class="messages-list">
            <div v-for="(m, idx) in messages" :key="idx" class="message-item" :class="`message-${m.role}`">
              <div class="message-bubble">
                <div class="message-header">
                  <q-avatar size="32px" :color="m.role === 'user' ? 'primary' : 'secondary'" text-color="white">
                    <q-icon :name="m.role === 'user' ? 'person' : 'smart_toy'" />
                  </q-avatar>
                  <span class="text-caption text-grey-7 q-ml-sm">{{ m.time }}</span>
                </div>
                <div class="message-text">{{ m.text }}</div>

                <div v-if="m.role === 'assistant' && m.sources?.length" class="message-sources">
                  <q-chip v-for="(s, i) in m.sources" :key="i" size="sm" clickable @click="openUrl(s)">
                    <q-icon name="link" size="xs" class="q-mr-xs" />
                    {{ shorten(s) }}
                  </q-chip>
                </div>
              </div>
            </div>
          </div>
        </q-card-section>

        <q-separator />

        <!-- Input Section -->
        <q-card-section class="chat-input">
          <q-banner v-if="!sessionId" rounded class="bg-orange-1 text-orange-9 q-mb-md" dense>
            <template v-slot:avatar>
              <q-icon name="info" />
            </template>
            Primero investiga un tema
          </q-banner>

          <q-banner v-if="error" rounded class="bg-red-1 text-red-9 q-mb-md" dense>
            <template v-slot:avatar>
              <q-icon name="error" />
            </template>
            {{ error }}
          </q-banner>

          <div class="row q-col-gutter-sm">
            <div class="col-12 col-sm-8">
              <q-input v-model.trim="question" type="textarea" autogrow outlined placeholder="Escribe tu pregunta..."
                :disable="!sessionId || loadingAsk" @keyup.ctrl.enter="onAsk" />
            </div>
            <div class="col-12 col-sm-4">
              <q-select v-model="topK" :options="[2, 3, 4, 5, 6, 8, 10]" label="Top K" outlined dense
                :disable="!sessionId || loadingAsk" />
              <q-btn label="Enviar" color="primary" class="full-width q-mt-sm" unelevated :loading="loadingAsk"
                :disable="!sessionId || !question || loadingAsk" @click="onAsk" />
            </div>
          </div>
        </q-card-section>
      </q-card>
    </div>
  </q-page>
</template>

<script setup>
import { ref } from 'vue'
import axios from 'axios'

const API = 'http://localhost:8000' // cambia si tu backend está en otro host/puerto

const topic = ref('')
const question = ref('')
const topK = ref(4)

const loadingResearch = ref(false)
const loadingAsk = ref(false)
const error = ref('')

const sessionId = ref('')
const sessionTopic = ref('')
const chunksIndexed = ref(null)
const sourceUrl = ref('')

const messages = ref([]) // { role: 'user'|'assistant', text, time, sources? }

function nowStr() {
  const d = new Date()
  return d.toLocaleString()
}

function shorten(url) {
  try {
    const u = new URL(url)
    return u.host + u.pathname
  } catch {
    return url.length > 40 ? url.slice(0, 40) + '...' : url
  }
}

function openUrl(url) {
  window.open(url, '_blank', 'noreferrer')
}

async function onResearch() {
  error.value = ''
  if (!topic.value) return

  loadingResearch.value = true
  try {
    const { data } = await axios.post(`${API}/research`, {
      topic: topic.value,
      max_chunks: 30,
      chunk_size: 500,
      chunk_overlap: 100,
    })

    sessionId.value = data.session_id
    sessionTopic.value = data.topic
    chunksIndexed.value = data.chunks_indexed
    sourceUrl.value = data.source

    // opcional: limpia chat y pregunta cuando cambias tema
    messages.value = []
    question.value = ''
  } catch (e) {
    error.value = e?.response?.data?.detail || e.message || 'Error en /research'
  } finally {
    loadingResearch.value = false
  }
}

async function onAsk() {
  error.value = ''
  if (!sessionId.value || !question.value) return

  // agrega mensaje del usuario
  messages.value.push({
    role: 'user',
    text: question.value,
    time: nowStr(),
  })

  loadingAsk.value = true
  const q = question.value
  question.value = ''

  try {
    const { data } = await axios.post(`${API}/ask`, {
      session_id: sessionId.value,
      question: q,
      top_k: topK.value,
    })

    messages.value.push({
      role: 'assistant',
      text: data.answer,
      time: nowStr(),
      sources: data.sources || [],
    })
  } catch (e) {
    const msg = e?.response?.data?.detail || e.message || 'Error en /ask'
    error.value = msg

    messages.value.push({
      role: 'assistant',
      text: `⚠️ ${msg}`,
      time: nowStr(),
      sources: [],
    })
  } finally {
    loadingAsk.value = false
  }
}
</script>

<style scoped lang="scss">
.rag-demo-page {
  background: linear-gradient(135deg, #758ae4 0%, #b69dce 100%);
  min-height: 100vh;
}

.page-container {
  max-width: 900px;
  margin: 0 auto;
  padding: 16px;
}

.page-header {
  color: white;
  margin-bottom: 24px;
  padding: 16px 8px;
}

.research-card,
.chat-card {
  border-radius: 16px;
  margin-bottom: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.session-info {
  .info-chip {
    background: #f5f5f5;
    border-radius: 8px;
    padding: 8px 12px;
    display: flex;
    align-items: center;
  }
}

.chat-card {
  display: flex;
  flex-direction: column;
  max-height: calc(100vh - 400px);
  min-height: 500px;
}

.chat-messages {
  flex: 1;
  overflow-y: auto;
  max-height: 500px;
}

.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding: 48px 16px;
}

.messages-list {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.message-item {
  display: flex;

  &.message-user {
    justify-content: flex-end;

    .message-bubble {
      background: #e3f2fd;
      border-radius: 16px 16px 4px 16px;
    }
  }

  &.message-assistant {
    justify-content: flex-start;

    .message-bubble {
      background: #f5f5f5;
      border-radius: 16px 16px 16px 4px;
    }
  }
}

.message-bubble {
  max-width: 80%;
  padding: 12px 16px;
}

.message-header {
  display: flex;
  align-items: center;
  margin-bottom: 8px;
}

.message-text {
  white-space: pre-wrap;
  word-wrap: break-word;
}

.message-sources {
  margin-top: 8px;
  display: flex;
  flex-wrap: wrap;
  gap: 4px;
}

@media (max-width: 600px) {
  .page-container {
    padding: 8px;
  }

  .message-bubble {
    max-width: 90%;
  }
}
</style>
