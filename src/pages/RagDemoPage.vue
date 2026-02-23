<template>
  <q-page class="q-pa-md" style="max-width: 900px; margin: 0 auto;">
    <div class="text-h5 q-mb-sm">RAG Demo (Tema → Pregunta)</div>
    <div class="text-caption text-grey-7 q-mb-md">
      Backend: FastAPI (localhost:8000). Primero crea una sesión con un tema, luego pregunta.
    </div>

    <!-- Tema -->
    <q-card class="q-mb-md">
      <q-card-section>
        <div class="row items-end q-col-gutter-md">
          <div class="col-12 col-md-8">
            <q-input v-model.trim="topic" label="Tema a investigar (Wikipedia)"
              placeholder="Ej: Roman Empire, Napoleon, Machine learning..." outlined dense :disable="loadingResearch"
              @keyup.enter="onResearch" />
          </div>

          <div class="col-12 col-md-4">
            <q-btn label="Investigar" color="primary" class="full-width" :loading="loadingResearch"
              :disable="!topic || loadingResearch" @click="onResearch" />
          </div>
        </div>

        <q-separator class="q-my-md" />

        <div class="row q-col-gutter-md">
          <div class="col-12 col-md-6">
            <q-banner rounded class="bg-grey-2">
              <div class="text-subtitle2">Sesión</div>
              <div class="text-body2">
                <div><b>session_id:</b> <span class="text-mono">{{ sessionId || "—" }}</span></div>
                <div><b>tema:</b> {{ sessionTopic || "—" }}</div>
                <div><b>chunks:</b> {{ chunksIndexed ?? "—" }}</div>
              </div>
            </q-banner>
          </div>

          <div class="col-12 col-md-6">
            <q-banner rounded class="bg-grey-2">
              <div class="text-subtitle2">Fuente</div>
              <div class="text-body2">
                <div v-if="sourceUrl">
                  <a :href="sourceUrl" target="_blank" rel="noreferrer">{{ sourceUrl }}</a>
                </div>
                <div v-else>—</div>
              </div>
            </q-banner>
          </div>
        </div>
      </q-card-section>
    </q-card>

    <!-- Pregunta -->
    <q-card>
      <q-card-section>
        <div class="text-subtitle1 q-mb-sm">Preguntar</div>

        <q-input v-model.trim="question" type="textarea" autogrow outlined label="Pregunta"
          placeholder="Escribe una pregunta técnica..." :disable="!sessionId || loadingAsk" @keyup.ctrl.enter="onAsk" />

        <div class="row items-center q-mt-sm q-col-gutter-md">
          <div class="col-12 col-md-4">
            <q-select v-model="topK" :options="[2, 3, 4, 5, 6, 8, 10]" label="Top K (contextos)" outlined dense
              :disable="!sessionId || loadingAsk" />
          </div>

          <div class="col-12 col-md-8">
            <div class="row justify-end q-gutter-sm">
              <q-btn label="Limpiar chat" flat color="grey-8"
                :disable="messages.length === 0 || loadingAsk || loadingResearch" @click="messages = []" />
              <q-btn label="Preguntar" color="primary" :loading="loadingAsk"
                :disable="!sessionId || !question || loadingAsk" @click="onAsk" />
            </div>
          </div>
        </div>

        <q-banner v-if="!sessionId" rounded class="bg-orange-1 text-orange-10 q-mt-md">
          Primero crea una sesión con <b>Investigar</b> (tema).
        </q-banner>

        <q-banner v-if="error" rounded class="bg-red-1 text-red-10 q-mt-md">
          {{ error }}
        </q-banner>

        <q-separator class="q-my-md" />

        <!-- Chat -->
        <div v-if="messages.length === 0" class="text-grey-7">
          Aquí aparecerán tus preguntas y respuestas.
        </div>

        <div v-else class="q-gutter-md">
          <q-card v-for="(m, idx) in messages" :key="idx" flat bordered class="q-pa-md">
            <div class="row items-start q-col-gutter-md">
              <div class="col-auto">
                <q-avatar :icon="m.role === 'user' ? 'person' : 'smart_toy'" />
              </div>
              <div class="col">
                <div class="text-subtitle2 q-mb-xs">
                  {{ m.role === "user" ? "Tú" : "Asistente" }}
                  <span class="text-caption text-grey-7 q-ml-sm">{{ m.time }}</span>
                </div>
                <div class="text-body2" style="white-space: pre-wrap;">{{ m.text }}</div>

                <div v-if="m.role === 'assistant' && m.sources?.length" class="q-mt-sm">
                  <div class="text-caption text-grey-8 q-mb-xs">Sources</div>
                  <div class="row q-col-gutter-sm">
                    <div class="col-12">
                      <q-chip v-for="(s, i) in m.sources" :key="i" clickable @click="openUrl(s)">
                        {{ shorten(s) }}
                      </q-chip>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </q-card>
        </div>
      </q-card-section>
    </q-card>
  </q-page>
</template>

<script setup>
import { ref } from "vue";
import axios from "axios";

const API = "http://localhost:8000"; // cambia si tu backend está en otro host/puerto

const topic = ref("");
const question = ref("");
const topK = ref(4);

const loadingResearch = ref(false);
const loadingAsk = ref(false);
const error = ref("");

const sessionId = ref("");
const sessionTopic = ref("");
const chunksIndexed = ref(null);
const sourceUrl = ref("");

const messages = ref([]); // { role: 'user'|'assistant', text, time, sources? }

function nowStr() {
  const d = new Date();
  return d.toLocaleString();
}

function shorten(url) {
  try {
    const u = new URL(url);
    return u.host + u.pathname;
  } catch {
    return url.length > 40 ? url.slice(0, 40) + "..." : url;
  }
}

function openUrl(url) {
  window.open(url, "_blank", "noreferrer");
}

async function onResearch() {
  error.value = "";
  if (!topic.value) return;

  loadingResearch.value = true;
  try {
    const { data } = await axios.post(`${API}/research`, {
      topic: topic.value,
      max_chunks: 30,
      chunk_size: 500,
      chunk_overlap: 100
    });

    sessionId.value = data.session_id;
    sessionTopic.value = data.topic;
    chunksIndexed.value = data.chunks_indexed;
    sourceUrl.value = data.source;

    // opcional: limpia chat y pregunta cuando cambias tema
    messages.value = [];
    question.value = "";
  } catch (e) {
    error.value = e?.response?.data?.detail || e.message || "Error en /research";
  } finally {
    loadingResearch.value = false;
  }
}

async function onAsk() {
  error.value = "";
  if (!sessionId.value || !question.value) return;

  // agrega mensaje del usuario
  messages.value.push({
    role: "user",
    text: question.value,
    time: nowStr()
  });

  loadingAsk.value = true;
  const q = question.value;
  question.value = "";

  try {
    const { data } = await axios.post(`${API}/ask`, {
      session_id: sessionId.value,
      question: q,
      top_k: topK.value
    });

    messages.value.push({
      role: "assistant",
      text: data.answer,
      time: nowStr(),
      sources: data.sources || []
    });
  } catch (e) {
    const msg = e?.response?.data?.detail || e.message || "Error en /ask";
    error.value = msg;

    messages.value.push({
      role: "assistant",
      text: `⚠️ ${msg}`,
      time: nowStr(),
      sources: []
    });
  } finally {
    loadingAsk.value = false;
  }
}
</script>

<style scoped>
.text-mono {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
}
</style>
