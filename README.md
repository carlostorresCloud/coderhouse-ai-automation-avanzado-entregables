# Checkpoint 4 — Sincronización del Cerebro Agéntico con Ecosistemas de Negocio

Workflow de automatización construido en **n8n** que conecta una casilla de correo de soporte (Gmail), un CRM (HubSpot o Salesforce) y un canal de equipo (Slack), aplicando principios de gobernanza, control de errores y *Human-in-the-loop* sobre un agente de IA.

## 📌 Descripción general

Este proyecto simula, a escala reducida, el ecosistema de negocio trabajado en la clase en vivo (Shopify → ERP → Slack → Klaviyo). En este caso, un correo entrante de un cliente es clasificado y respondido por un Agente de IA, pero antes de llegar al cliente o impactar en el CRM, el workflow aplica una serie de controles de seguridad y calidad de datos.

**Rol de cada conector:**

| Pieza del vivo | Herramienta usada | Rol que cumple |
|---|---|---|
| ERP / Shopify | HubSpot o Salesforce | Fuente única de verdad del cliente (CRM) |
| Casilla de soporte | Gmail | Entrada y salida de comunicación con el cliente |
| Canal del equipo | Slack | Notificación al equipo de operaciones |

## 🧠 Arquitectura del workflow

```
[Trigger: Email entrante]
        │
        ▼
① [IF — ¿es auto-reply?] ── Sí ─▶ (Stop: corta el bucle infinito)
        │ No
        ▼
   [AI Agent: clasifica / redacta]
        │
        ▼
④ [Set — limpia y valida el payload]
        │
        ▼
② [Look up — ¿el contacto ya existe en el CRM?]
   ┌────┴─────┐
   Sí          No
   ▼            ▼
[Update]    [Create contacto]
        │
        ▼
③ [Create Draft — borrador para aprobación humana (HITL)]
```

## 🔑 Los 4 nodos clave que evalúa la rúbrica

1. **① IF anti auto-reply** — ubicado inmediatamente después del trigger de entrada de correo. Escanea el asunto del email e ignora automáticamente mensajes con etiquetas como `Auto-reply`, `Out of office`, `Undeliverable` o remitentes `no-reply@`, evitando que el workflow entre en un bucle infinito de auto-respuestas.

2. **② Look up antes del Create** — antes de crear un contacto nuevo en el CRM, el workflow busca si ya existe un registro con ese email. Esto previene el **Error 409** (duplicados de contactos).

3. **③ Create Draft (guardrail Human-in-the-loop)** — el nodo de salida de correo está configurado exclusivamente para crear un **borrador** en Gmail, nunca para enviar el mensaje de forma automática. Esto garantiza que un humano revise y apruebe la respuesta generada por la IA antes de su emisión final.

4. **④ Set de limpieza de payload** — previo a los conectores de mensajería masiva (Slack), un nodo `Set` deja únicamente los campos necesarios (`From`, `Subject`, `BodyText`) y valida que el email no esté vacío, evitando el **Error 400** por payload mal formado y evitando saturar el canal con objetos binarios pesados.

## 🔒 Seguridad y permisos

- Autenticación de cada conector (Gmail, CRM, Slack) mediante **OAuth2**.
- Scopes de permisos acotados al **mínimo privilegio** necesario para la función operativa (solo lectura/escritura de los campos que el flujo realmente usa).

## 🧪 Testing

Cada nodo fue validado individualmente mediante `Test step` / `Execute Workflow` en n8n para certificar la estabilidad de la interconexión antes de la exportación final.

## 📂 Contenido del repositorio

```
├── checkpoint4_nombre_apellido.json   # Workflow exportado desde n8n (Workflow > Download)
└── README.md                          # Este archivo
```

## 🚀 Cómo importar el workflow

1. Abrir n8n.
2. Ir a **Workflows > Import from File**.
3. Seleccionar el archivo `checkpoint4_nombre_apellido.json`.
4. Configurar las credenciales (Gmail, CRM, Slack) con las propias del entorno de destino.
5. Activar el workflow o ejecutarlo en modo test.

## 🛠️ Stack utilizado

- **n8n** (orquestación del workflow)
- **Gmail** (casilla de soporte al cliente)
- **HubSpot / Salesforce** (CRM)
- **Slack** (canal de equipo)
- **AI Agent** (clasificación y redacción de respuestas)

---
Proyecto realizado como parte del curso de Coderhouse — *IA Generativa Aplicada / Automatización con Agentes*.
