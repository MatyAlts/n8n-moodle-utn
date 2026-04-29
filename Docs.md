# Sistema RAG + Human-in-the-Loop para Foros Moodle UTN

**Fecha de última actualización:** 2026-04-29  
**Plataforma n8n:** https://belmontelucero-n8n.326kz3.easypanel.host  
**Moodle:** https://tup.sied.utn.edu.ar

---

## Visión General

Sistema de 3 workflows n8n que permite:

1. **Indexar** el material de estudio (PDFs de Google Drive) en un vector store para RAG
2. **Monitorear** los foros de Moodle y notificar por Telegram cuando hay consultas sin respuesta
3. **Responder** esas consultas con asistencia de IA, con un humano que aprueba, edita o descarta la respuesta antes de publicar

El modelo de interacción es **B1 (human-in-the-loop)**: la IA genera una propuesta, el humano la revisa vía Telegram y decide qué hacer.

---

## Stack Tecnológico

| Componente | Tecnología | Detalle |
|---|---|---|
| Orquestación | n8n (self-hosted) | easypanel |
| Embeddings | Gemini `gemini-embedding-001` | 3072 dimensiones |
| Vector store | Pinecone | Índice `moodle-utn`, dims=3072, región us-east-1 |
| LLM | Gemini Flash (`gemini-3.1-flash-lite-preview`) | RAG Chain |
| Fuente de datos | Google Drive | PDFs por unidad temática |
| Notificaciones | Telegram Bot | Un bot para el proyecto |
| Publicación | Moodle Mobile Web Services | REST API deshabilitada, MWS activo |

---

## Arquitectura del Sistema

```
Google Drive (PDFs)
        |
        v
[Workflow 1: Ingesta RAG]
        |
        v
Pinecone (moodle-utn, 3072 dims)
        |
        v
[Workflow 2: Monitor Foros]
Moodle -> Telegram (notificación + botón)
        |
        v
[Workflow 3: Responder Foros]
Telegram Trigger -> RAG -> Preview -> Publicar en Moodle
```

---

## Workflow 1 — Ingesta RAG

**ID:** `d6iN5IGIsknraCGw`  
**Trigger:** Manual (o schedule)  
**Propósito:** Indexar PDFs del material de estudio en Pinecone para usar como contexto RAG

### Flujo de nodos

```
Manual Trigger
  → List PDFs in Unit (Google Drive API, por unidad)
  → Filter Already Ingested (Code — static data)
  → Download PDF (Google Drive)
  → Extract from File (operation: pdf)
  → Filter Empty (text.length > 100)
  → Default Data Loader
  → Recursive Character Text Splitter
  → Pinecone Insert (sin clearNamespace)
       ↑
  Gemini Embeddings (gemini-embedding-001)
```

### Ingesta incremental

Se usa `$getWorkflowStaticData('global')` para trackear los IDs de Google Drive ya procesados. Si un archivo ya fue indexado, se saltea.

```javascript
// Nodo: Filter Already Ingested
const staticData = $getWorkflowStaticData('global');
const processedIds = new Set(staticData.processedFileIds || []);
const items = $input.all();
const newItems = items.filter(item => !processedIds.has(item.json.id));
newItems.forEach(item => processedIds.add(item.json.id));
staticData.processedFileIds = Array.from(processedIds);
return newItems;
```

> **Importante:** `clearNamespace: true` fue eliminado para no borrar todo el índice en cada ejecución.

### Estructura de Google Drive

| Unidad | PDFs aproximados |
|---|---|
| Unidad 1 | ~5 |
| Unidad 2 | ~5 |
| Unidad 3 | ~5 |
| Unidad 4 | ~5 |
| Unidad 5 | ~5 |

---

## Workflow 2 — Monitor Foros

**ID:** `pAPlJkDBP8ErZeiM`  
**Trigger:** Schedule (polling periódico)  
**Propósito:** Detectar consultas de alumnos sin respuesta y notificar por Telegram

### Flujo de nodos

```
Schedule Trigger
  → Get Token (Moodle MWS login)
  → Buscar Consultas Sin Respuesta (mod_forum_get_forum_discussions)
  → Filtrar Ya Notificadas (Code — static data)
  → Notificar por Telegram (un mensaje por discusión + botón inline)
```

### Filtro de ya notificadas

```javascript
// Nodo: Filtrar Ya Notificadas
const staticData = $getWorkflowStaticData('global');
const notifiedIds = new Set(staticData.notifiedDiscussionIds || []);
const items = $input.all();
const newItems = items.filter(item => !notifiedIds.has(item.json.discussionId));
newItems.forEach(item => notifiedIds.add(item.json.discussionId));
staticData.notifiedDiscussionIds = Array.from(notifiedIds);
return newItems;
```

### Mensaje de Telegram

Cada discusión genera un mensaje con:
- Nombre del foro y título de la consulta
- Botón inline **🤖 Generar respuesta IA** con `callback_data: "generate:{discussionId}"`

#### Configuración correcta del nodo Telegram (CRÍTICO)

```javascript
// replyMarkup e inlineKeyboard van al nivel RAÍZ, NO dentro de additionalFields
parameters: {
  chatId: '1415649706',
  text: '={{ ... }}',
  replyMarkup: 'inlineKeyboard',       // ← NIVEL RAÍZ
  inlineKeyboard: {                     // ← NIVEL RAÍZ
    rows: [{ row: { buttons: [{
      text: '🤖 Generar respuesta IA',
      additionalFields: { callback_data: '={{ "generate:" + $json.discussionId }}' }
    }] } }]
  },
  additionalFields: { parse_mode: 'HTML', appendAttribution: false }
}
```

---

## Workflow 3 — Responder Foros

**ID:** `WcbnXeAhpCPJKELZ`  
**Trigger:** Telegram Trigger (events: `callback_query`, `message`)  
**Propósito:** Generar respuestas con RAG y publicarlas en Moodle con aprobación humana

### Flujo general

```
Telegram Trigger
  → Parse Event (Code)
  → Route Action (Switch — 5 cases)
       ├── case 0: generate  → Ack → Get Token → Fetch Posts → Prepare Question → RAG Chain → Store Response → Send Preview
       ├── case 1: publish   → Get Stored Response → Ack → Get Token → Post to Moodle → Confirm Published
       ├── case 2: edit      → Store Pending Edit → Ack → Ask for Text
       ├── case 3: discard   → Ack → Confirm Discard
       └── case 4: text_reply → Get Token → Post to Moodle Reply → Clear Pending → Confirm Reply
```

### Nodo: Parse Event

Distingue entre `callback_query` (botones) y `message` (texto del usuario):

```javascript
const item = $input.first().json;
const staticData = $getWorkflowStaticData('global');

if (item.callback_query) {
  const [action, discussionId] = (item.callback_query.data || '').split(':');
  return [{ json: {
    action,
    discussionId: parseInt(discussionId),
    callbackQueryId: item.callback_query.id,
    chatId: String(item.callback_query.message?.chat?.id || '1415649706')
  }}];
}

if (item.message?.text) {
  const pending = staticData.pendingEdit;
  if (pending) {
    return [{ json: {
      action: 'text_reply',
      discussionId: pending.discussionId,
      postId: pending.postId,
      text: item.message.text,
      chatId: String(item.message.chat?.id || '1415649706')
    }}];
  }
}

return [{ json: { action: 'ignore' } }];
```

### Nodo: Fetch Discussion Posts

Llama a `mod_forum_get_discussion_posts` de Moodle MWS.

> **CRÍTICO:** El parámetro se llama **`discussionid`** (no `discussid`). Usar el nombre incorrecto genera un error `invalidparameter`.

La respuesta tiene un array `posts` donde `posts[0]` es la pregunta principal:
- `posts[0].id` → `postId` (se usa como padre para la respuesta)
- `posts[0].message` → HTML con el contenido de la pregunta
- `posts[0].subject` → título

### Nodo: Store Response

Convierte la respuesta del LLM de markdown a HTML, y pre-computa los strings de callback para los botones:

```javascript
function formatInline(text) {
  return text
    .replace(/\*\*(.+?)\*\*/g, '<b>$1</b>')
    .replace(/__(.+?)__/g, '<b>$1</b>')
    .replace(/`([^`]+)`/g, '<code>$1</code>');
}

function markdownToHtml(text) {
  const lines = text.split('\n');
  const result = [];
  let inOl = false, inUl = false;

  for (const line of lines) {
    const olMatch = line.match(/^\d+\.\s+(.*)/);
    const ulMatch = line.match(/^[-*]\s+(.*)/);

    if (olMatch) {
      if (inUl) { result.push('</ul>'); inUl = false; }
      if (!inOl) { result.push('<ol>'); inOl = true; }
      result.push('<li>' + formatInline(olMatch[1]) + '</li>');
    } else if (ulMatch) {
      if (inOl) { result.push('</ol>'); inOl = false; }
      if (!inUl) { result.push('<ul>'); inUl = true; }
      result.push('<li>' + formatInline(ulMatch[1]) + '</li>');
    } else {
      if (inOl) { result.push('</ol>'); inOl = false; }
      if (inUl) { result.push('</ul>'); inUl = false; }
      result.push(line.trim() === '' ? '<br>' : '<p>' + formatInline(line) + '</p>');
    }
  }

  if (inOl) result.push('</ol>');
  if (inUl) result.push('</ul>');
  return result.join('\n');
}

// Guardar en static data la versión HTML (para Moodle)
staticData.pendingResponses[String(discussionId)] = { text: htmlResponse, postId, subject };

// Pre-computar callbacks (CRÍTICO — ver sección de gotchas)
return [{ json: {
  preview,
  discussionId,
  chatId,
  cbPublish: 'publish:' + discussionId,
  cbEdit:    'edit:'    + discussionId,
  cbDiscard: 'discard:' + discussionId
} }];
```

### Nodo: Send Preview

Muestra la respuesta generada con tres botones de acción:

```javascript
// Botones referenciando strings pre-computados (CRÍTICO)
{ text: '✅ Publicar', additionalFields: { callback_data: '={{ $json.cbPublish }}' } }
{ text: '✏️ Editar',   additionalFields: { callback_data: '={{ $json.cbEdit }}' } }
{ text: '❌ Descartar', additionalFields: { callback_data: '={{ $json.cbDiscard }}' } }
```

### Nodo: Post to Moodle

Publica la respuesta vía `mod_forum_add_discussion_post`:
- `postid`: ID del post padre (primer post de la discusión)
- `message`: HTML generado (con `messageformat=1`)
- `subject`: asunto de la respuesta

### Static data del Workflow 3

```javascript
// Estructura que se persiste entre ejecuciones
staticData.pendingResponses = {
  "9390": { text: "<p>Respuesta HTML...</p>", postId: 123, subject: "Re:" }
}
staticData.pendingEdit = {
  discussionId: "9390",
  postId: 123,
  chatId: "1415649706"
}
```

---

## Autenticación Moodle Mobile Web Services

```
POST https://tup.sied.utn.edu.ar/login/token.php
Content-Type: application/x-www-form-urlencoded

username=DNI_MOODLE
password=PASSWORD_MOODLE
service=moodle_mobile_app
```

Respuesta: `{ "token": "abc123..." }`

El token se usa en todas las llamadas a:
```
https://tup.sied.utn.edu.ar/webservice/rest/server.php
  ?wstoken={token}
  &wsfunction={función}
  &moodlewsrestformat=json
```

> **Nota:** La REST API de Moodle (`/webservice/rest/server.php` con `wstoken` de REST) está **deshabilitada** en este servidor. Mobile Web Services es un canal **separado e independiente** que SÍ está habilitado. No confundirlos.

---

## Gotchas y Problemas Conocidos

### 1. n8n no evalúa expresiones complejas en `callback_data` de botones Telegram

n8n **NO** evalúa en runtime expresiones como `={{ "publish:" + $json.discussionId }}` cuando están dentro de `additionalFields.callback_data` de un botón inline keyboard. Envía el string literal de la expresión.

**Solución:** Pre-computar los strings en un Code node anterior y referenciarlos con expresiones simples:
```javascript
// Code node (Store Response): pre-computar
return [{ json: { cbPublish: 'publish:' + discussionId } }];

// Telegram node (Send Preview): referencia simple
{ callback_data: '={{ $json.cbPublish }}' }  // ✅ funciona
{ callback_data: '={{ "publish:" + $json.discussionId }}' }  // ❌ no se evalúa
```

### 2. `replyMarkup` e `inlineKeyboard` deben ir al nivel raíz del nodo Telegram

Si se ponen dentro de `additionalFields`, los botones no aparecen en el mensaje.

### 3. Parámetro Moodle: `discussionid` no `discussid`

La función `mod_forum_get_discussion_posts` requiere el parámetro `discussionid`. Usar `discussid` genera `invalidparameter` aunque el valor llegue correctamente.

### 4. El LLM genera markdown, Moodle necesita HTML

- Moodle acepta HTML con `messageformat=1`. Si se envía markdown crudo, se muestra sin formatear.
- Telegram preview: solo inline formatting (`<b>`, `<i>`, `<code>`) — no renderiza bien `<p>` ni listas HTML.
- Por eso hay dos versiones: `htmlResponse` (para Moodle) y `telegramPreview` (solo inline, para el preview).

### 5. Static data no editable via UI en esta versión de n8n

El menú del workflow no muestra "Edit Static Data". Para resetear durante pruebas, agregar temporalmente al Code node:
```javascript
staticData.notifiedDiscussionIds = [];  // Workflow 2
staticData.processedFileIds = [];       // Workflow 1
staticData.pendingResponses = {};       // Workflow 3
```
Ejecutar una vez y luego eliminar esas líneas.

### 6. Gemini Embeddings: dimensión real es 3072

El nodo `embeddingsGoogleGemini` en n8n produce vectores de 3072 dimensiones con `gemini-embedding-001`. El índice Pinecone debe crearse con `dims=3072`.

> Historial: el proyecto usó Mistral Embed (1024 dims) temporalmente por rate limiting de Gemini. Desde que se obtuvo API key paga, se volvió a Gemini y se recreó el índice.

---

## Estado Actual del Sistema (2026-04-29)

| Flujo | Estado |
|---|---|
| Workflow 1: Ingesta incremental | ✅ Funciona |
| Workflow 2: Notificación por discusión + botón | ✅ Funciona |
| Workflow 3: Generar respuesta IA | ✅ Funciona |
| Workflow 3: Publicar en Moodle (HTML formateado) | ✅ Funciona |
| Workflow 3: Descartar respuesta | ✅ Funciona |
| Workflow 3: Editar + publicar texto manual | ✅ Funciona |

---

## Pendiente

- Probar el flujo de edición manual completo (✏️ Editar → escribir texto → verificar publicación en Moodle)
- Considerar un schedule para Workflow 2 (actualmente manual)
- Evaluar agregar más unidades al índice a medida que avanza la cursada
