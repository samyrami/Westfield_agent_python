# API de Agentes — Guía de integración para el front

Documentación de los endpoints expuestos en AWS bajo la ruta del gateway:

```
/api/v1/universities/*/agents/*
```

Los `*` son variables de la URL:

```
/api/v1/universities/{university_code}/agents/{agent_id}/health
/api/v1/universities/{university_code}/agents/{agent_id}/chat
```

| Variable | Descripción | Ejemplo |
|---|---|---|
| `university_code` | Slug de la universidad (tenant). Se mapea a su carpeta en S3 — una universidad nueva no requiere cambios de código. | `westfield` |
| `agent_id` | Identificador único del agente dentro de esa universidad. | `maia` |

**Base URL:** el dominio del gateway de AWS del ambiente correspondiente, por ejemplo:

```
https://<gateway-aws>/api/v1/universities/{university_code}/agents/{agent_id}/...
```

## Generalidades

- **Formato:** JSON en request y response (`Content-Type: application/json`).
- **Sin streaming:** no hay SSE ni WebSockets; cada turno de chat es un request/response normal.
- **Autenticación:** hoy no es obligatoria. CORS acepta los headers `Content-Type` y `Authorization` y los métodos `GET` y `POST`, sin credenciales (cookies).
- **Backend stateless:** el servidor **no guarda la conversación**. El front debe conservar el historial y enviarlo completo en cada turno (ver [Manejo de la conversación](#manejo-de-la-conversación-desde-el-front)).
- **Errores:** todos los errores tienen el mismo shape:

```json
{ "error": "descripción del problema" }
```

---

## 1. Health del agente

```
GET /api/v1/universities/{university_code}/agents/{agent_id}/health
```

Fuerza la carga del agente en el runtime y reporta su estado. Útil para:
- Verificar que el `university_code` y el `agent_id` existen antes de abrir el chat.
- Precalentar el agente (la primera carga lee S3; las siguientes usan caché).

### Respuesta `200 OK`

```json
{
  "university": "westfield",
  "agent_id": "maia",
  "ok": true,
  "agent_name": "Maia",
  "degraded": false,
  "llm_provider": "openai",
  "llm_model": "gpt-4o-mini",
  "prompt_id": "maia-entrevista-v1",
  "vector_store_id": "vs-maia-01",
  "chunks": 124
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `university` | `string` | Eco del `university_code` de la URL. |
| `agent_id` | `string` | Eco del `agent_id` de la URL. |
| `ok` | `boolean` | Siempre `true` si responde 200 (si falla, responde 404/503). |
| `agent_name` | `string` | Nombre visible del agente (para mostrar en UI). |
| `degraded` | `boolean` | `true` si el agente cargó pero con capacidades reducidas (ej. sin vector store). Funciona igual, pero sin RAG. |
| `llm_provider` | `string` | Proveedor del LLM (ej. `"openai"`). |
| `llm_model` | `string` | Modelo configurado. |
| `prompt_id` | `string` | Identificador del prompt del agente. |
| `vector_store_id` | `string \| null` | Vector store del agente, o `null` si no usa RAG. |
| `chunks` | `number` | Cantidad de chunks cargados del vector store (0 si no tiene). |

### Errores

| Código | Cuándo | Body |
|---|---|---|
| `404` | `university_code` no existe | `{ "error": "..." }` |
| `404` | `agent_id` no existe para esa universidad | `{ "error": "..." }` |
| `503` | El agente existe pero falló al cargar (config corrupta, prompt ausente, proveedor desconocido) | `{ "error": "..." }` |

### Ejemplo `curl`

```bash
curl https://<gateway-aws>/api/v1/universities/westfield/agents/maia/health
```

---

## 2. Chat (un turno de conversación)

```
POST /api/v1/universities/{university_code}/agents/{agent_id}/chat
```

Envía un turno del usuario y devuelve la respuesta del agente.

### Request body

```json
{
  "conversation_id": "conv-8f3a2b",
  "user_id": "user-123",
  "message": "Hola, quiero información sobre el programa de admisión",
  "history": [
    { "role": "user", "content": "Hola" },
    { "role": "assistant", "content": "¡Hola! ¿En qué puedo ayudarte?" }
  ],
  "state": null
}
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `conversation_id` | `string` | ✅ | Identificador de la conversación. Lo genera el front (ej. un UUID al abrir el chat) y se repite en todos los turnos. |
| `user_id` | `string` | ✅ | Identificador del usuario final. |
| `message` | `string` | No (default `""`) | Mensaje nuevo del usuario. Máximo efectivo: **4.000 caracteres** (el backend trunca el excedente). |
| `history` | `ChatTurn[]` | No (default `[]`) | Historial previo en orden cronológico, **sin incluir** el `message` actual. El backend solo usa los **últimos 30 turnos**. |
| `state` | `object \| null` | No | Dict libre para agentes con mecánica de turnos (ej. Maia: `current_question`, `turns_for_current_question`). Si el agente lo usa, el front debe reenviar el `state` que considere vigente. Si no, omitirlo o enviar `null`. |

Cada elemento de `history` (`ChatTurn`):

| Campo | Tipo | Valores |
|---|---|---|
| `role` | `string` | `"user"` o `"assistant"` |
| `content` | `string` | Texto del turno |

### Respuesta `200 OK`

```json
{
  "agent_id": "maia",
  "conversation_id": "conv-8f3a2b",
  "message": "Claro, el proceso de admisión consta de tres pasos...",
  "structured": null,
  "sources": [
    { "doc_title": "Guía de admisiones 2026", "doc_tag": "admisiones", "similarity": 0.87 }
  ],
  "fallback": false,
  "rag_used": true
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `agent_id` | `string` | Eco del agente que respondió. |
| `conversation_id` | `string` | Eco del `conversation_id` enviado. |
| `message` | `string` | Texto de la respuesta del agente. **Es lo que se muestra en el chat.** |
| `structured` | `object \| null` | Solo viene si el agente está configurado con `response_format=json` (respuesta estructurada). Para agentes de texto normal siempre es `null`. |
| `sources` | `SourceRef[]` | Documentos usados como contexto RAG (solo metadata, sin texto). Puede usarse para mostrar "Fuentes" en la UI. Vacío si no hubo RAG. |
| `fallback` | `boolean` | `true` si el LLM falló o no está disponible y `message` contiene el mensaje de respaldo del agente. Ver tabla de estados. |
| `rag_used` | `boolean` | `true` si se incorporó contexto del vector store en la respuesta. |

Cada elemento de `sources` (`SourceRef`):

| Campo | Tipo | Descripción |
|---|---|---|
| `doc_title` | `string` | Título del documento fuente. |
| `doc_tag` | `string` | Etiqueta/categoría del documento. |
| `similarity` | `number` | Similitud del chunk con la consulta (0–1). |

### Errores y rate limit

| Código | Cuándo | Body |
|---|---|---|
| `404` | `university_code` o `agent_id` no existen | `{ "error": "..." }` |
| `422` | Body inválido (falta `conversation_id`/`user_id`, tipos incorrectos) | Error de validación de FastAPI |
| `429` | Rate limit: máximo **12 requests por 60 segundos** por IP + universidad + agente | `{ "error": "Demasiadas solicitudes. Espera un minuto y vuelve a intentar." }` |
| `503` | El agente falló al cargar | `{ "error": "..." }` |

### Ejemplo `curl`

```bash
curl -X POST \
  https://<gateway-aws>/api/v1/universities/westfield/agents/maia/chat \
  -H "Content-Type: application/json" \
  -d '{
    "conversation_id": "conv-8f3a2b",
    "user_id": "user-123",
    "message": "Hola, quiero información sobre admisiones",
    "history": []
  }'
```

---

## Tabla de estados que el front debe contemplar

| Situación | HTTP | Señal | Qué hacer en la UI |
|---|---|---|---|
| Respuesta normal | 200 | `fallback=false` | Mostrar `message` (y `sources` si aplica). |
| LLM caído / sin API key | 200 | `fallback=true` | Mostrar `message` igual (es el mensaje de respaldo del agente). Opcional: marcar visualmente que el servicio está degradado. **No es un error HTTP.** |
| Sin contexto RAG (vector store roto o sin matches) | 200 | `rag_used=false`, `sources=[]` | Nada especial; la respuesta sigue siendo válida. **No es un error.** |
| Universidad o agente inexistente | 404 | `{"error": ...}` | Mostrar "agente no disponible"; revisar la URL construida. |
| Demasiadas solicitudes | 429 | `{"error": ...}` | Bloquear el input y sugerir reintentar en ~1 minuto. |
| Agente mal configurado | 503 | `{"error": ...}` | Mostrar error de servicio; reintentar más tarde no ayuda hasta que se corrija la config. |

## Manejo de la conversación desde el front

El backend **no persiste nada**: el front es dueño del estado de la conversación.

1. Al abrir el chat, generar un `conversation_id` (ej. `crypto.randomUUID()`).
2. En cada turno, enviar el `message` nuevo + todo el `history` acumulado (el backend usa los últimos 30 turnos).
3. Al recibir la respuesta, agregar al historial local el turno del usuario y el del asistente:
   `[...history, {role: "user", content: message}, {role: "assistant", content: respuesta.message}]`.
4. Si el agente usa `state` (mecánica de turnos), conservarlo y reenviarlo según la lógica acordada con ese agente.

## Tipos TypeScript (copiar y pegar)

```typescript
export type ChatRole = "user" | "assistant";

export interface ChatTurn {
  role: ChatRole;
  content: string;
}

export interface ChatRequest {
  conversation_id: string;
  user_id: string;
  message?: string;            // default "" — máx. 4.000 caracteres
  history?: ChatTurn[];        // default [] — el backend usa los últimos 30 turnos
  state?: Record<string, unknown> | null;
}

export interface SourceRef {
  doc_title: string;
  doc_tag: string;
  similarity: number;          // 0–1
}

export interface ChatResponse {
  agent_id: string;
  conversation_id: string;
  message: string;
  structured: Record<string, unknown> | null; // solo agentes con response_format=json
  sources: SourceRef[];
  fallback: boolean;           // true → message es el mensaje de respaldo
  rag_used: boolean;
}

export interface AgentHealthResponse {
  university: string;
  agent_id: string;
  ok: boolean;
  agent_name: string;
  degraded: boolean;
  llm_provider: string;
  llm_model: string;
  prompt_id: string;
  vector_store_id: string | null;
  chunks: number;
}

export interface ApiError {
  error: string;
}
```

## Ejemplo de cliente `fetch`

```typescript
const BASE_URL = "https://<gateway-aws>"; // dominio del gateway de AWS

export async function sendChatTurn(
  universityCode: string,
  agentId: string,
  body: ChatRequest,
): Promise<ChatResponse> {
  const res = await fetch(
    `${BASE_URL}/api/v1/universities/${universityCode}/agents/${agentId}/chat`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(body),
    },
  );

  if (!res.ok) {
    const err: ApiError = await res.json().catch(() => ({ error: res.statusText }));
    if (res.status === 429) {
      throw new Error(err.error); // "Demasiadas solicitudes..." → reintentar en ~1 min
    }
    throw new Error(err.error ?? `HTTP ${res.status}`);
  }

  return res.json();
}

export async function getAgentHealth(
  universityCode: string,
  agentId: string,
): Promise<AgentHealthResponse> {
  const res = await fetch(
    `${BASE_URL}/api/v1/universities/${universityCode}/agents/${agentId}/health`,
  );
  if (!res.ok) {
    const err: ApiError = await res.json().catch(() => ({ error: res.statusText }));
    throw new Error(err.error ?? `HTTP ${res.status}`);
  }
  return res.json();
}
```

---

*Fuente: `src/westfield_agent_back_python/entrypoints/api.py` (rutas y errores) y `src/westfield_agent_back_python/domain/chat.py` (modelos `ChatRequest` / `ChatResponse`).*
