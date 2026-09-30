# Entregable Clase 4 — Integraciones externas con OAuth
 
Workflow de automatización construido en **n8n** que conecta **Gmail**, un **Agente de IA (OpenAI)**, **Google Sheets** (como CRM personal) y **Slack**, todas las integraciones autenticadas mediante **OAuth2**.
 
## 📌 Descripción general
 
Este flujo automatiza el seguimiento de correos de procesos de búsqueda de empleo (por ejemplo, notificaciones de **Workday**). Cuando llega un correo, el workflow filtra los mensajes automáticos, usa un Agente de IA para extraer los datos relevantes del contacto/empresa y un resumen del mensaje, y luego actualiza un CRM propio en Google Sheets (planilla *"Búsqueda de empleo"*). Antes de responder al remitente, el mensaje se deja como **borrador** en Gmail para aprobación humana, y el equipo es notificado por **Slack**.
 
> Este flujo aplica sobre un caso propio (seguimiento de postulaciones laborales) los mismos conceptos trabajados en la clase en vivo: fuente única de verdad, human-in-the-loop y datos limpios antes de notificar.
 
## 🧠 Arquitectura del workflow
 
```
[Gmail Trigger] — poll cada hora, filtra remitentes que contengan "workday"
        │
        ▼
① [Es autoreply? — IF]
   ¿Subject contiene "auto-reply" / "out of office" / "undeliverable"
    o From contiene "noreply"?
        │
   Sí ──▶ [terminar flujo — NoOp]  (corta el bucle infinito)
        │ No
        ▼
   [AI Lead classificator]  (Langchain Chain LLM + modelo gpt-4o-mini)
        │  usa [Structured Output Parser] → { nombre, correo, resumen }
        ▼
   [Normalizar payload — Set]
        │  deja los campos: from, contacto, respuesta_ia
        ▼
② [CRM - Buscar Contacto — Google Sheets: appendOrUpdate]
   busca/matchea por columna "Candidato" en la planilla "Búsqueda de empleo"
        │
        ▼
   [IF - Contacto existe?]
   ¿existe el campo "Candidato" en el resultado?
   ┌────┴─────┐
   Sí          No
   ▼            ▼
③ [Hacer borrador   [CRM - Crear contacto — Google Sheets: append]
   - Human in the      agrega Candidato / Empresa / Notas / Fase = "En proceso"
   the loop]  (Gmail)
   │
   ▼
[Notificar al equipo de soporte — Slack #equipo-de-soporte]
```
 
## 🔑 Nodos clave del workflow
 
1. **① `Es autoreply?` (IF anti auto-reply)** — justo después del Gmail Trigger. Evalúa con un `OR` de 4 condiciones si el asunto contiene `auto-reply`, `out of office`, `undeliverable`, o si el remitente contiene `noreply`. Si se cumple alguna, el flujo termina en el nodo `terminar flujo` (NoOp), evitando procesar respuestas automáticas y cortando el bucle infinito.
2. **`AI Lead classificator` + `Model` + `Structured Output Parser`** — un nodo *Chain LLM* de Langchain (modelo `gpt-4o-mini`, temperatura 0.3) que analiza el remitente, asunto y snippet del correo, extrae el nombre del contacto y genera un resumen. El `Structured Output Parser` fuerza la respuesta a un JSON con el esquema `{ nombre, correo, resumen }`.
3. **`Normalizar payload` (Set)** — limpia la salida de la IA y la deja en tres campos simples (`from`, `contacto`, `respuesta_ia`), evitando pasar objetos anidados o datos innecesarios a los nodos siguientes.
4. **② `CRM - Buscar Contacto` (Google Sheets — `appendOrUpdate`)** — busca en la planilla *"Búsqueda de empleo"* si ya existe una fila para ese `Candidato`; si existe la actualiza, si no, la deja preparada para la verificación del siguiente paso. Cumple el rol de "fuente única de verdad" antes de decidir qué hacer.
5. **`IF - Contacto existe?`** — valida si el resultado de la búsqueda trae el campo `Candidato`. Según el resultado, deriva el flujo hacia la rama de respuesta (contacto ya registrado) o hacia la creación de un registro nuevo.
6. **`CRM - Crear contacto` (Google Sheets — `append`)** — si el contacto es nuevo, agrega una fila con `Candidato`, `Empresa`, `Notas` (resumen generado por la IA) y `Fase = "En proceso"`.
7. **③ `Hacer borrador - Human in the loop` (Gmail, `resource: draft`)** — en lugar de enviar la respuesta automáticamente, el nodo crea un **borrador** en Gmail con el resumen generado por la IA, garantizando que una persona revise y apruebe el contenido antes de enviarlo.
8. **`Notificar al equipo de soporte` (Slack — canal `equipo-de-soporte`)** — envía un mensaje al canal del equipo indicando que hay un nuevo borrador para revisión humana, junto con el remitente y el resumen generado por la IA.
## 🔒 Seguridad y autenticación
 
Todas las integraciones están conectadas mediante **OAuth2**:
- **Gmail account** (trigger y creación de borradores).
- **OpenAI account** (modelo `gpt-4o-mini` para el Agente de IA).
- **Google Sheets account** (CRM — planilla *"Búsqueda de empleo"*).
- **Slack account** (canal `equipo-de-soporte`).
## 📂 Contenido del repositorio
 
```
├── checkpoint4_Carlos_Torres.json   # Workflow exportado desde n8n
└── README.md                                                     # Este archivo
```


## ✅ Configuración y validación de OAuth2 (Gmail y Google Sheets)
 
Como uso una instancia selfhosteada de n8n, las credenciales de Google no se conectan con un clic: hay que crear mi propia app OAuth en **Google Cloud Platform (GCP)** y enlazarla a n8n, mediante los siguientes pasos:

### Paso 1 — Crear el proyecto en Google Cloud
 
1. Entrar a [Google Cloud Console](https://console.cloud.google.com/) y crear un proyecto nuevo (por ejemplo, `n8n-oauth-entregable`).
2. Verificar que el proyecto quede seleccionado en el selector superior.

### Paso 2 — Habilitar las APIs necesarias
 
En **APIs y servicios > Biblioteca**, habilitar:
 
- **Gmail API**
- **Google Sheets API**

### Paso 3 — Configurar la pantalla de consentimiento OAuth
 
En **APIs y servicios > Pantalla de consentimiento de OAuth**:
 
1. Elegir el tipo de usuario (**Externo** para cuentas personales de Gmail).
2. Completar nombre de la app y correo de contacto.
3. Agregar los **scopes** necesarios (Gmail y Google Sheets).
4. En modo *Testing*, agregar mi cuenta en **Usuarios de prueba** (sin esto, Google bloquea el login con el error `access_denied`).


### Paso 4 — Crear las credenciales OAuth (Client ID y Client Secret)
 
En **APIs y servicios > Credenciales > Crear credenciales > ID de cliente de OAuth**:
 
1. Tipo de aplicación: **Aplicación web**.
2. En **URI de redireccionamiento autorizados**, pegar la *OAuth Redirect URL* que muestra n8n al crear la credencial



Evidencias de oauth funcionando en n8n

![ OAuth Client ID - gmail ](./screenshots/gmail-oauth2.png)

![ OAuth Client ID - sheets ](./screenshots/GoogleSheets-oauth2.png)

 
## 🚀 Cómo importar el workflow
 
1. Abrir n8n.
2. Ir a **Workflows > Import from File**.
3. Seleccionar el archivo `Entregable_clase_4_-_Integraciones_externas_con_oauth.json`.
4. Configurar las credenciales propias:
   - Gmail OAuth2(en gcp para la version local o selfhosteada de n8n).
   - OpenAI API.
   - Google Sheets OAuth2 (apuntando a una planilla propia con columnas `Candidato`, `Fase`, `Rol`, `Empresa`, `Notas`).
   - Slack API (seleccionando el canal de destino).
5. Activar el workflow o ejecutarlo en modo test (`Test step` / `Execute Workflow`) nodo por nodo.
## 🛠️ Stack utilizado
 
- **n8n** (orquestación del workflow)
- **Gmail** (trigger de entrada y creación de borradores)(oauth configurado en gcp)
- **OpenAI (gpt-4o-mini)** vía **Langchain** (clasificación y resumen del correo)
- **Google Sheets** (CRM personal de seguimiento de postulaciones)(oauth configurado en gcp)
- **Slack** (notificación al equipo)
---
 




