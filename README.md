# Chatbot WhatsApp + IA (n8n) 

> **Flujo:** `chatbot-v2`  
> **Qué hace:** Recibe mensajes de **WhatsApp** (texto, audio, imagen y PDF), usa **OpenAI** para transcribir/analisar/generar respuesta, puede **responder con texto o audio**, extrae texto de **PDF**, y consulta/crea datos en **Gmail / Google Calendar / Notion**.

---

## 👀 Demo / Capturas

> Deja aquí tus imágenes (diagramas, pantallas de n8n, ejemplo de conversación, etc.)

- **Arquitectura general**  
  `![Arquitectura](docs/arquitectura.png)`

- **Vista del workflow en n8n**  
  `![Workflow](docs/workflow.png)`

- **Ejemplos de uso (texto / audio / imagen / PDF)**  
  `![Ejemplos](docs/ejemplos.png)`

---

## ✨ Funcionalidades

- **Entrada por WhatsApp** (Webhook oficial de Meta): texto, **audio**, **imagen**, **PDF**.  
- **Audio → Texto** (OpenAI Whisper).  
- **Imagen → Descripción** (OpenAI Vision).  
- **PDF → Texto** (Extract From File).  
- **Respuesta**:
  - Texto en WhatsApp.
  - **Audio** en WhatsApp (Text‑to‑Speech con OpenAI).
- **Herramientas**:
  - **Gmail**: leer, borrar, obtener por MessageID, crear borradores y enviar mail.
  - **Google Calendar**: consultar eventos, crear eventos.
  - **Notion**: consultar y crear registros (finanzas, proyectos y tareas), y **update** de páginas.
- **Memoria de ventana**: contexto corto por contacto.

---

## 🧩 Principales nodos del flujo

- **WhatsApp Trigger** → enruta por tipo: `Switch "Input type"`.
- **Texto** → va directo al **Assistant Agent**.
- **Audio** → `Get Audio Url` → `Download Audio` → `Transcribe Audio` → **Assistant Agent**.
- **Imagen** → `Get Image Url` → `Download Image` → `Analyze Image` → **Assistant Agent**.
- **PDF** → `Only PDF File` → `Get File Url` → `Download File` → `Extract from File` → **Assistant Agent**.
- **Respuesta**:
  - Si el usuario mandó audio, se intenta **responder en audio** (`Generate Audio Response` → `Fix mimeType for Audio` → `Send audio`).
  - Si no, **texto** (`Send message`).
- **Herramientas del agente**: Gmail, Calendar, Notion, Embeddings + Vector Store.

---

## 🧰 Requisitos

- **n8n** ≥ 1.50 (recomendado)  
- Cuenta de **Meta for Developers** con **WhatsApp Business Platform** (número y webhook configurado).  
- API Key de **OpenAI**.  
- Credenciales OAuth para **Gmail** y **Google Calendar**.  
- Integración de **Notion** y bases de datos creadas:
  - _Financial Tracking_ (campos: **Descripcion** (title), **Categoria** (rich_text), **Monto** (number), **Fecha** (date), **Tipo** (rich_text)).
  - _Projects_ y _Tasks_ (nombres de tus DB tal como en el flujo).

---

## 📦 Instalación de n8n

### Opción A) Docker (recomendado)

```bash
docker run -it --rm   -p 5678:5678   -e N8N_HOST=localhost   -e N8N_PORT=5678   -e N8N_PROTOCOL=http   -e DB_SQLITE_VACUUM_ON_STARTUP=true   -e DB_SQLITE_POOL_SIZE=4   -v ~/.n8n:/home/node/.n8n   n8nio/n8n:latest
```

- **Nota**: `DB_SQLITE_POOL_SIZE` > 0 evita advertencias y mejora lecturas en paralelo.

### Opción B) Node.js (sin Docker)

```bash
# 1) Instalar n8n global
npm install -g n8n

# 2) Definir variables mínimas (ejemplo en Linux/macOS)
export DB_SQLITE_POOL_SIZE=4

# 3) Ejecutar n8n
n8n
```

n8n quedará en `http://localhost:5678`.

---

## 🔐 Variables de entorno (ejemplos)

> Ajusta según tu despliegue (Docker `-e` o archivo `.env` si usas n8n con dotenv).

```
# OpenAI
OPENAI_API_KEY=sk-...

# WhatsApp / Meta
META_APP_ID=...
META_APP_SECRET=...
WHATSAPP_TOKEN=...
WHATSAPP_VERIFY_TOKEN=...       # para la verificación del webhook
WHATSAPP_PHONE_NUMBER_ID=...

# Gmail & Calendar (OAuth)
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
GOOGLE_REDIRECT_URI=...

# Notion
NOTION_API_KEY=secret_...
```

---

## 🧭 Cómo **importar** este flujo en n8n

### Método 1: Importar desde archivo

1. Copia el JSON del flujo (lo que me pasaste).
2. Guarda como `chatbot-v2.json`.
3. En n8n: **Workflows → Import from File** → selecciona `chatbot-v2.json`.
4. Guarda y **Activa** el workflow.

### Método 2: Importar desde portapapeles

1. Copia **todo** el JSON del flujo.
2. En n8n: **Workflows → Import from Clipboard** → pega el JSON.
3. Guarda y **Activa** el workflow.

> **Ojo con las credenciales por nombre**  
> El flujo hace referencia a credenciales con nombres específicos. En **Credentials** crea o asigna:
>
> - **WhatsApp OAuth account** / **WhatsApp chat bot**
> - **OpenAi account**
> - **correo profesional** (Gmail OAuth2 y Google Calendar OAuth2)
> - **Notion account**
> - **chat bot** (HTTP Header Auth si lo usas para descargar media)
>
> Si usas **otros nombres**, entra a cada nodo y selecciona la credencial correcta.

---

## 🧩 Conectar WhatsApp (Meta)

1. En **developers.facebook.com** crea app → **WhatsApp**.  
2. Configura **Webhook**:
   - **Callback URL**: `https://TU_DOMINIO/webhook/whatsapp` (lo que n8n te indique en el nodo **WhatsApp Trigger**).
   - **Verify Token**: el mismo que pusiste en n8n/entorno.  
   - Suscribe el campo **messages**.
3. Asocia **Phone Number ID** al número.  
4. En el nodo **WhatsApp Trigger** confirma `webhookId` y credenciales.

> **Prueba rápida**: desde la **Consola de WhatsApp** de Meta envía un mensaje de prueba a tu número de “test”.

---

## 🧪 Pruebas de cada tipo

- **Texto**: envía cualquier texto → respuesta del **Assistant Agent** por **texto**.
- **Audio**: envía nota de voz → se transcribe y el bot **responde en audio** (TTS).
- **Imagen**: envía foto → se descarga → **Analyze Image** describe y responde.
- **PDF**: envía documento **PDF** → extrae contenido → responde.  
  - Si el documento **NO** es PDF → se envía “Sorry but you can only send PDF files”.

---

## ⚙️ Personalización rápida

- **Modelos OpenAI**: en `Analyze Image / Transcribe Audio / Chat Model / TTS` puedes cambiar `gpt-4o-mini`, `gpt-4o`, voz TTS, etc.
- **Respuesta por defecto** (nodo **Different input**): ajusta el texto para entradas no soportadas.
- **Memoria**: `Window Buffer Memory` por `wa_id` (contacto). Ajusta `contextWindowLength`.

---

## 🗂️ Estructura Notion esperada

- **Financial Tracking**:
  - `Descripcion` (title)
  - `Categoria` (rich_text)
  - `Monto` (number)
  - `Fecha` (date)
  - `Tipo` (rich_text)

- **Projects** y **Tasks**:
  - Usa los IDs/URLs de tus bases. En los nodos **Get many projects / Get many tasks** reemplaza por tus DB.

---

## 📧 Gmail / 📅 Calendar

- Acepta el **consentimiento OAuth** en n8n al probar los nodos.  
- Para **borrar** o **responder hilos**, primero usa “Get email by MessageID” y guarda el `MessageID` que el agente te pedirá.

---

## 🧯 Troubleshooting

- **El webhook de WhatsApp no dispara**  
  - Verifica **Callback URL / Verify Token** y que el **Trigger** esté **activo**.  
  - En Meta, en **Suscripciones** marca **messages**.

- **Descargas de media fallan**  
  - Revisa credenciales del nodo **HTTP Request** (header `Authorization: Bearer WHATSAPP_TOKEN` si aplica).  
  - Asegúrate de pasar la **URL** correcta (`mediaUrlGet`).

- **Audio no se reproduce**  
  - El nodo **Fix mimeType for Audio** ajusta `audio/mp3` → `audio/mpeg`. Confirma tipo de archivo.

- **Límites de SQLite**  
  - Exporta a una DB externa o usa `DB_SQLITE_POOL_SIZE=4`.

- **Scopes de Google**  
  - Reautoriza Gmail y Calendar si cambiaste scopes o cuentas.

---

## 🚀 Despliegue

- **Docker Compose** (ejemplo mínimo):

```yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n:latest
    ports:
      - '5678:5678'
    environment:
      - N8N_HOST=tu-dominio.com
      - N8N_PROTOCOL=https
      - DB_SQLITE_POOL_SIZE=4
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - WHATSAPP_TOKEN=${WHATSAPP_TOKEN}
      - WHATSAPP_VERIFY_TOKEN=${WHATSAPP_VERIFY_TOKEN}
      # ...agrega todas tus variables
    volumes:
      - ~/.n8n:/home/node/.n8n
```

- Coloca un **reverse proxy** (Caddy/NGINX) con HTTPS para exponer el webhook.

---

## 🗺️ Roadmap (ideas)

- Control de **roles/permitidos** por número.  
- **Logs** y métricas (por tipo de entrada).  
- Integración con **Drive/OneDrive** para almacenar media.  
- Persistencia de memoria por usuario (DB propia).

---

## 📄 Licencia

MIT (o la que definas).

---

## 🙌 Créditos

- Construido con **n8n**, **WhatsApp Business Platform** (Meta), **OpenAI**, **Gmail/Google Calendar**, **Notion**.

---

## 📎 Anexos

> Pega aquí el **JSON del flujo** o enlázalo como archivo `workflows/chatbot-v2.json` en el repo para importarlo desde archivo.

```text
(workflows/chatbot-v2.json)
```

---

### ✅ Checklist de publicación

- [ ] Variables de entorno configuradas.  
- [ ] Credenciales creadas en n8n con **los nombres esperados** o reasignadas en cada nodo.  
- [ ] Webhook de **WhatsApp** verificado y activo.  
- [ ] Notion: DBs creadas y **IDs/URLs** actualizados en nodos.  
- [ ] Gmail/Calendar con OAuth autorizado.  
- [ ] Pruebas de **texto / audio / imagen / PDF** realizadas.  

