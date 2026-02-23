# Rag Wiki App (app-rag-llm)

Aplicación de **RAG (Retrieval-Augmented Generation)** sobre Wikipedia. Permite investigar cualquier tema consultando Wikipedia, indexar el contenido en fragmentos (_chunks_) y luego hacer preguntas en lenguaje natural sobre ese tema. Las respuestas son generadas por un LLM usando únicamente el contexto recuperado.

## ¿Cómo funciona?

```
Usuario ──► [1. Tema] ──► POST /research ──► Wikipedia ──► Chunks indexados (sesión)
                                                                    │
Usuario ──► [2. Pregunta] ──► POST /ask ──► Top-K chunks ──► LLM ──► Respuesta + fuentes
```

1. **Investigar** – Se ingresa un tema (ej. `Roman Empire`, `Machine learning`). El backend llama a la API de Wikipedia, extrae el artículo, lo divide en chunks y los indexa en memoria bajo un `session_id`.
2. **Preguntar** – Con la sesión activa, se hacen preguntas libres. El backend recupera los _Top-K_ chunks más relevantes y los envía al LLM para generar una respuesta con sus fuentes.

## Tech Stack

| Capa               | Tecnología                 |
| ------------------ | -------------------------- |
| Frontend           | Vue 3 + Quasar v2 + Vite   |
| Estado             | Pinia                      |
| HTTP               | Axios                      |
| Backend (separado) | FastAPI – `localhost:8000` |

## Endpoints del backend

| Método | Ruta        | Descripción                                                    |
| ------ | ----------- | -------------------------------------------------------------- |
| `POST` | `/research` | Crea sesión: descarga y fragmenta el artículo de Wikipedia     |
| `POST` | `/ask`      | Responde una pregunta usando los chunks indexados de la sesión |

### Body `/research`

```json
{
  "topic": "Roman Empire",
  "max_chunks": 30,
  "chunk_size": 500,
  "chunk_overlap": 100
}
```

### Body `/ask`

```json
{
  "session_id": "<uuid>",
  "question": "¿Cuándo cayó el Imperio Romano?",
  "top_k": 4
}
```

## Instalación y desarrollo

```bash
# Instalar dependencias
npm install

# Iniciar en modo desarrollo (hot-reload)
npm run dev
# o
quasar dev
```

> Asegúrate de tener el backend FastAPI corriendo en `http://localhost:8000` antes de usar la app.

## Scripts disponibles

```bash
npm run lint      # Verificar código con ESLint
npm run format    # Formatear con Prettier
npm run build     # Build de producción
quasar build      # Build de producción (alternativa)
```

## Configuración

Ver [Configuring quasar.config.js](https://v2.quasar.dev/quasar-cli-vite/quasar-config-js).

Para cambiar la URL del backend, edita la constante `API` en `src/pages/RagDemoPage.vue`:

```js
const API = 'http://localhost:8000' // cambia si tu backend está en otro host/puerto
```
