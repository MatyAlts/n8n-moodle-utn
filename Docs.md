# Sistema RAG Moodle UTN — Documentación

## Descripción general

Sistema automático que monitorea foros de Moodle (UTN), detecta preguntas sin respuesta, notifica por Telegram y genera respuestas usando IA con RAG (Retrieval-Augmented Generation) basado en el material de cada materia.

---

## Arquitectura

```
Moodle Foros
     ↓ (cada 30 min)
Monitor Workflow
     ↓ (Telegram)
Tutor recibe notificación con botón "Generar respuesta IA"
     ↓ (click)
Responder Workflow
     ↓
Pinecone RAG (namespace por materia) + Gemini Flash
     ↓
Respuesta generada → Tutor revisa y publica en Moodle
```

---

## Materias configuradas

| Materia | courseId | Namespace Pinecone | Foro IDs (cmids) |
|---------|----------|--------------------|------------------|
| Prog 1 | 38 | `prog1` | 11156, 11201, 11226, 11181, 11248, 11273, 11300, 13747, 11322, 11346 |
| Prog 2 | 42 | `prog2` | 12121, 12146, 12170, 12194, 12218, 12238, 12260, 12280, 12299, 12312 |
| Prog 3 | 44 | `prog3` | 14152, 13169, 13185, 13207, 13226, 13273, 13296, 13316, 13336, 13351 |

---

## Workflows n8n

**Host:** https://belmontelucero-n8n.326kz3.easypanel.host

### Ingesta RAG
- **ID:** `eIEIBbeBEGRjVwsy`
- **Trigger:** Form manual con dropdown de materia (prog1 / prog2 / prog3)
- **Función:** Recorre carpetas de Google Drive, descarga PDFs, extrae texto, vectoriza e inserta en Pinecone en el namespace correspondiente
- **Static data:** Lleva registro de archivos ya ingestados por materia (incremental)

### Monitor PROG1
- **ID:** `9IUOJ3vsUlJEDO8t`
- **Trigger:** Schedule cada 30 minutos
- **courseId:** 38 | **namespace:** prog1
- **Notifica a:** `1415649706`

### Monitor PROG2
- **ID:** `XFMyik6K8DjFmWqk`
- **Trigger:** Schedule cada 30 minutos
- **courseId:** 42 | **namespace:** prog2
- **Notifica a:** `1415649706`

### Monitor PROG3
- **ID:** `WYxCJPR293XMPmMI`
- **Trigger:** Schedule cada 30 minutos
- **courseId:** 44 | **namespace:** prog3
- **Notifica a:** `1415649706` (tutor principal) y `1631091257` (tutor 2)

### Responder
- **ID:** `qQRdcFrIXdycRKbU`
- **Trigger:** Webhook de Telegram (callback_query del botón)
- **Función:** Extrae namespace del callback, busca contexto en Pinecone, genera respuesta con Gemini Flash via RAG Chain

---

## Pinecone

- **Index:** `moodle-utn`
- **Namespaces:** `prog1`, `prog2`, `prog3`
- **Modelo de embeddings:** Gemini Embeddings
- **Records cargados:** Prog1 = 546 records, Prog2 = 672, Prog3 = 559

---

## Google Drive — Carpetas de material

| Carpeta | ID |
|---------|----|
| RAG root | `18YbblCchD7-OehJfejXnT_OjBs1z0fw_` |
| Prog1 | `1d4yiQzrvgRDZPUc9aNg7S3PzP0cB-JaT` |
| Prog2 | `1Hot5A5uxBXvUnuNPWtOyVeFe3HbTXeLH` |
| Prog3 | `1Liqve8pM_vhXMN-rGc_vcEZ9lAq_XyQB` |

---

## Telegram

- **Bot:** usado para recibir notificaciones y callbacks de generación
- **Chat IDs configurados:**
  - `1415649706` — tutor principal (recibe Prog1, Prog2 y Prog3)
  - `1631091257` — tutor Prog3 (solo recibe Prog3)

### Formato del callback data
```
generate:{discussionId}:{namespace}
```
Ejemplo: `generate:12345:prog2`

El Responder extrae el namespace del callback para hacer el retrieval en el namespace correcto de Pinecone.

---

## Flujo de datos detallado

### Monitor → Responder (via Telegram)
1. Monitor detecta discusión con `numreplies === 0`
2. Arma mensaje con título, autor, cuerpo y link a Moodle
3. Envía notificación Telegram con botón inline cuyo `callback_data` = `generate:{discussionId}:{namespace}`
4. Tutor hace click → Telegram envía callback al webhook del Responder
5. **Parse Event** extrae `action`, `discussionId`, `namespace` del callback
6. **Prepare Question** obtiene el texto de la discusión desde Moodle API
7. **Pinecone Retrieve** busca chunks relevantes en el namespace de la materia
8. **RAG Chain** (Gemini Flash) genera la respuesta usando el contexto recuperado
9. Respuesta se envía al tutor por Telegram para revisión

---

## Moodle API

- **Host:** `https://tup.sied.utn.edu.ar`
- **Autenticación:** token via `/login/token.php` (usuario/contraseña)
- **Endpoints usados:**
  - `mod_forum_get_forums_by_courses` — lista foros de un curso
  - `mod_forum_get_forum_discussions` — lista discusiones de un foro
  - `mod_forum_get_forum_discussion_posts` — obtiene posts de una discusión

---

## Pendiente

- [ ] Ingestar material de Prog2 en Pinecone (carpeta Drive configurada)
- [ ] Ingestar material de Prog3 en Pinecone (carpeta Drive configurada)
- [ ] Activar Monitor PROG2 y PROG3 desde el UI de n8n
- [ ] Test end-to-end con pregunta real en foro de Prog1
- [ ] Automatizar ingesta (actualmente es manual via Form Trigger)
