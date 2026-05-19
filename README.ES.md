# 🏥 Omnichannel AI Scheduler — Clínica Estética

> Sistema omnicanal de gestión de citas con IA para clínicas de estética. Construido con n8n, Gemini AI, WhatsApp Business API, Instagram DM, Facebook Messenger, Google Calendar, Google Sheets, Gmail y **GoHighLevel (LeadConnector)**. Automatiza reservas, cancelaciones, reprogramaciones y escalado humano sin intervención manual.

[![N8N](https://img.shields.io/badge/N8N-Workflow-orange?style=flat-square)](https://n8n.io)
[![Gemini AI](https://img.shields.io/badge/Gemini-AI-4285F4?style=flat-square&logo=google)](https://ai.google.dev)
[![WhatsApp](https://img.shields.io/badge/WhatsApp-Business_API-25D366?style=flat-square&logo=whatsapp)](https://developers.facebook.com/docs/whatsapp)
[![Instagram](https://img.shields.io/badge/Instagram-DM-E4405F?style=flat-square&logo=instagram)](https://developers.facebook.com/docs/instagram)
[![Facebook](https://img.shields.io/badge/Facebook-Messenger-1877F2?style=flat-square&logo=facebook)](https://developers.facebook.com/docs/messenger-platform)
[![GoHighLevel](https://img.shields.io/badge/GoHighLevel-CRM-8A2BE2?style=flat-square)](https://www.gohighlevel.com)
[![Licencia: MIT](https://img.shields.io/badge/Licencia-MIT-22c55e?style=flat-square)](LICENSE)
[![Estado](https://img.shields.io/badge/Estado-Producción-22c55e?style=flat-square)](https://github.com/alejandro-orbis)

---

## 📚 Documentación

| Idioma | Archivo |
|--------|---------|
| Inglés | [README.md](README.md) |
| Español | [README.ES.md](README.ES.md) |

---

## 🎯 ¿Qué hace?

Sistema omnicanal de gestión de citas para clínicas de estética. Los pacientes pueden interactuar a través de **WhatsApp, Instagram o Facebook Messenger** para:

| Función | Descripción |
|---------|-------------|
| **Agendar cita** | El paciente solicita una cita, el bot recopila datos (nombre, teléfono, edad) y muestra disponibilidad real del Google Calendar |
| **Confirmar cita** | Tras elegir horario, el paciente confirma y el bot crea el evento en el calendario |
| **Reprogramar cita** | El paciente puede cambiar su cita a otro horario disponible |
| **Cancelar cita** | El paciente cancela su cita existente |
| **Escalado a humano** | Si el paciente necesita hablar con una persona, el bot deriva al staff y se silencia automáticamente |
| **Triaje médico** | Verifica edad y condiciones de salud antes de agendar |
| **Reset de conversación** | El usuario puede escribir `reset` para limpiar todo el contexto |

---

## ✨ Características principales

| Característica | Descripción |
|----------------|-------------|
| **Omnicanal real** | WhatsApp Business API, Instagram DM, Facebook Messenger |
| **IA conversacional multi-turno** | Contexto persistente entre sesiones usando caché + Google Sheets |
| **Máquina de estados** | 9 etapas (inicio, triaje, recopilando_nombre, recopilando_telefono, mostrando_slots, confirmando_cita, agendado, cancelado, escalar) |
| **Disponibilidad real** | Consulta Google Calendar en tiempo real, muestra huecos libres |
| **Protección anti-doble confirmación** | Evita duplicados si el usuario responde "sí" varias veces |
| **Triaje médico** | Verifica edad (≥18) y condiciones (embarazo, anticoagulantes, alergias) |
| **Intervención humana** | Silencia el bot durante 30 minutos cuando un humano toma control |
| **Reset completo** | Comando `reset` limpia caché y contexto en Sheets |
| **Lead scoring** | Puntúa cada lead según intención (HOT 70-100, WARM 40-69, COOL 15-39, COLD 0-14) |
| **Rate limiting** | 8 mensajes por minuto por conversación |
| **Filtrado de eco** | Previene bucles con mensajes propios del bot |

---

## 🏗️ Arquitectura

```
Webhook (WhatsApp / Instagram / Facebook)
│
└── Parsear Webhook (normaliza formatos)
│
└── Rate Limiter (8 msg/minuto)
│
└── Resolver Identidad Cross-Canal (patient_id + conversation_id únicos)
│
└── Intervencion Activa IF (human_takeover = TRUE silencia el bot)
│
└── Motor Estado Multi-turno
│
├── AGENDAR → Consultar disponibilidad Calendar → Crear evento
│
├── CANCELAR → Cancelar evento Calendar
│
├── CAMBIAR → Mostrar nuevos slots → Reprogramar
│
├── ESCALAR → Activar human_takeover + Email al staff
│
└── RESPONDER → Gemini IA clasifica intención
│
└── Preparar Respuesta → Guardar contexto
│
├── Enviar mensaje (WhatsApp / Instagram / Facebook)
│
├── Guardar en Sheets (Contexto, Pacientes, CRM_Inbox)
│
├── Actualizar Metrics (leads, citas, escalados)
│
└── Sincronizar GHL (contacto, oportunidad, nota)
```
---

## 🧠 Etapas del Motor de Estado

| Etapa | Descripción | Siguiente si ✅ | Siguiente si ❌ |
|-------|-------------|-----------------|-----------------|
| **inicio** | Clasifica intención con Gemini | `AGENDAR` → `triaje` | `OTRO` → responde y espera |
| **triaje** | Pregunta edad, detecta contraindicaciones | Edad ≥18 → `recopilando_nombre` | Contraindicación → `escalar` |
| **recopilando_nombre** | Pide nombre completo | Nombre válido → `recopilando_telefono` | No es nombre → repite |
| **recopilando_telefono** | Pide teléfono, normaliza a 34XXXXXXXXX | Teléfono válido → `mostrando_slots` | No válido → repite |
| **mostrando_slots** | Muestra hasta 6 huecos libres del Calendar | Elige número → `confirmando_cita` | "cambiar" → otros slots |
| **confirmando_cita** | Resumen + "¿Confirmas? SÍ/NO" | "SÍ" → `agendado` | "NO" → `mostrando_slots` |
| **agendado** | Crea evento en Calendar + notifica | "cancelar" → `confirmando_cancelacion` | "gracias" → despedida |
| **confirmando_cancelacion** | "¿Confirmas cancelación? SÍ/NO" | "SÍ" → `cancelado` | "NO" → `agendado` |
| **cancelado** | Elimina evento, limpia metadatos | Nueva cita → `mostrando_slots` | Despedida → `inicio` |
| **escalar** | Deriva a atención humana | Email al staff, bot silenciado 30min | Staff resuelve → `inicio` |

---

## 🛠️ Stack tecnológico

| Herramienta | Uso |
|-------------|-----|
| [N8N](https://n8n.io) | Orquestación de flujos y automatización |
| [Gemini AI](https://ai.google.dev) | Clasificación de intención |
| [WhatsApp Business API](https://developers.facebook.com/docs/whatsapp) | Mensajería entrante y saliente |
| [Instagram DM API](https://developers.facebook.com/docs/instagram) | Mensajería por Instagram |
| [Facebook Messenger API](https://developers.facebook.com/docs/messenger-platform) | Mensajería por Facebook |
| [Google Calendar](https://calendar.google.com) | Gestión de citas y disponibilidad |
| [Google Sheets](https://sheets.google.com) | Persistencia de contexto, pacientes, CRM y métricas |
| [Gmail](https://gmail.com) | Notificaciones al staff |
| [GoHighLevel / LeadConnector](https://gohighlevel.com) | CRM externo |

---

## 📸 Capturas de pantalla

| Vista general del workflow | Dashboard de métricas | Ejemplo WhatsApp | Ejemplo Instagram |
|---|---|---|---|
| ![Overview](assets/n8n/workflow_overview.png) | ![Dashboard](assets/dashboard/dashboard_overview.png) | ![WhatsApp](assets/conversations/conversation_example_whatsapp.png) | ![Instagram](assets/conversations/conversation_example_instagram.png) |

| Ejemplo Facebook | Email de confirmación | GHL Contactos | GHL Oportunidades |
|---|---|---|---|
| ![Facebook](assets/conversations/conversation_example_facebook.png) | ![Email](assets/email/email_confirmation.png) | ![GHL Contact](assets/ghl/ghl_contact_in_ghl.png) | ![GHL Opportunity](assets/ghl/ghl_opportunity_in_ghl.png) |

| Google Sheets Contexto | Google Calendar Evento | Nodo Gemini IA | Nodos GHL en n8n |
|---|---|---|---|
| ![Contexto](assets/google_sheets/google_sheets_contexto.png) | ![Calendar](assets/google_calendar/google_calendar_appointment.png) | ![Gemini](assets/n8n/n8n_gemini_node.png) | ![GHL Nodes](assets/ghl/ghl_workflow_nodes.png) |

---

## 📁 Estructura del proyecto

```
Webhook (WhatsApp / Instagram / Facebook)
│
└── Parsear Webhook (normaliza formatos)
│
└── Rate Limiter (8 msg/minuto)
│
└── Resolver Identidad Cross-Canal (patient_id + conversation_id únicos)
│
└── Intervencion Activa IF (human_takeover = TRUE silencia el bot)
│
└── Motor Estado Multi-turno
│
├── AGENDAR → Consultar disponibilidad Calendar → Crear evento
│
├── CANCELAR → Cancelar evento Calendar
│
├── CAMBIAR → Mostrar nuevos slots → Reprogramar
│
├── ESCALAR → Activar human_takeover + Email al staff
│
└── RESPONDER → Gemini IA clasifica intención
│
└── Preparar Respuesta → Guardar contexto
│
├── Enviar mensaje (WhatsApp / Instagram / Facebook)
│
├── Guardar en Sheets (Contexto, Pacientes, CRM_Inbox)
│
├── Actualizar Metrics (leads, citas, escalados)
│
└── Sincronizar GHL (contacto, oportunidad, nota)
```

---

## 🚀 Guía de configuración

### 1. Requisitos previos

- Instancia de N8N (self-hosted v2.10+ o N8N Cloud)
- Cuenta Meta for Developers con acceso a:
  - WhatsApp Business API
  - Instagram Graph API
  - Facebook Messenger Platform
- Cuenta Google (Sheets + Calendar + Gmail)
- Cuenta Google AI Studio (API key de Gemini)
- GoHighLevel / LeadConnector (opcional)

### 2. Configurar credenciales en N8N

| Credencial | Dónde obtenerla |
|------------|------------------|
| `GEMINI_API_KEY` | [aistudio.google.com](https://aistudio.google.com) |
| `META_ACCESS_TOKEN` | Meta for Developers → Tu App → WhatsApp → Configuración de API |
| `META_PAGE_ACCESS_TOKEN` | Meta for Developers → Tu App → Instagram → Page Access Token |
| `WHATSAPP_PHONE_NUMBER_ID` | Meta for Developers → WhatsApp → Configuración de API |
| `META_IG_PAGE_ID` | Meta for Developers → Instagram → Cuentas vinculadas |
| `META_FB_PAGE_ID` | Meta for Developers → Facebook → Página |
| `META_BOT_PSID` | ID de la página (evita eco del bot) |
| Google Sheets OAuth | OAuth integrado de N8N |
| Google Calendar OAuth | OAuth integrado de N8N |
| Gmail OAuth | OAuth integrado de N8N |
| `GHL_ACCESS_TOKEN` | GoHighLevel → Configuración → API → Token de acceso |
| `GHL_LOCATION_ID` | GoHighLevel → Configuración → Ubicación |

### 3. Subir la plantilla de Google Sheets

1. Sube `Omnichannel AI Scheduler — Medical Clinic.xlsx` a Google Drive
2. Copia el **Sheet ID** de la URL:
https://docs.google.com/spreadsheets/d/TU_SHEET_ID_AQUI/edit

3. Actualiza el `documentId` en todos los nodos de Google Sheets del workflow

### 4. Importar workflows a N8N

En N8N: menú `...` → **Importar desde archivo** → selecciona:

| Workflow | Archivo | Método | Propósito |
|----------|---------|--------|-----------|
| Principal | `workflow/omnichannel-ai-scheduler.json` | POST | Gestionar mensajes entrantes |
| Verificación | `workflow/meta-webhook-verification.json` | GET | Verificar webhook con Meta |

### 5. Configurar webhooks de Meta

| Canal | URL del Webhook |
|-------|-----------------|
| WhatsApp | `https://tu-n8n.com/webhook/omnichannel-webhook` |
| Instagram | `https://tu-n8n.com/webhook/omnichannel-webhook` |
| Facebook | `https://tu-n8n.com/webhook/omnichannel-webhook` |

**Token de verificación:** Configura `META_VERIFY_TOKEN` en tus variables de entorno. Este mismo token debe configurarse en Meta Business.

### 6. Activar el workflow

Cambia el toggle del workflow a **Activo** en N8N.

---

## 📊 Estructura de Google Sheets

### Hoja Contexto

| Columna | Descripción |
|---------|-------------|
| `conversation_id` | ID único por conversación |
| `from` | ID/teléfono del remitente |
| `channel` | whatsapp / instagram / facebook / test |
| `stage` | Etapa actual (inicio, triaje, agendado, etc.) |
| `history` | Últimos 20 turnos de conversación |
| `metadata` | JSON con nombre, teléfono, edad, tratamientos_interes, aptitudVerificada |
| `cita_confirmada` | TRUE / FALSE |

### Hoja Pacientes

| Columna | Descripción |
|---------|-------------|
| `patient_id` | ID único del paciente |
| `telefono_normalizado` | 34XXXXXXXXX — clave de búsqueda cross-canal |
| `nombre` | Nombre completo |
| `canal_origen` | Primer canal que usó |
| `citas_confirmadas` | Contador de citas completadas |

### Hoja CRM_Inbox

| Columna | Descripción |
|---------|-------------|
| `tipo_evento` | CITA_CONFIRMADA / ESCALADO / CITA_CANCELADA |
| `paciente_nombre` | Nombre del paciente |
| `servicio` | Tratamiento solicitado |
| `calendar_event_id` | ID del evento en Google Calendar |

### Hoja Metrics

| Columna | Descripción |
|---------|-------------|
| `total_leads` | Total de conversaciones |
| `leads_cualificados` | Intención de agendar |
| `citas_agendadas` | Citas confirmadas |
| `escalados` | Casos derivados a humano |

### Hoja Intervenciones

| Columna | Descripción |
|---------|-------------|
| `conversation_id` | ID de la conversación intervenida |
| `activa` | TRUE = bot silenciado / FALSE = bot activo |

---

## 🔧 Nodos clave del workflow

| Nodo | Función |
|------|---------|
| `Parsear Webhook` | Normaliza payloads de WhatsApp, Instagram, Facebook y Test |
| `Rate Limiter` | 8 mensajes por minuto por conversación |
| `Resolver Identidad Cross-Canal` | Genera patient_id y conversation_id únicos |
| `Intervencion Activa IF` | Silencia el bot si human_takeover = TRUE (30 min) |
| `Control Lecturas Sheets` | Decide si usar Sheets o caché |
| `Calcular Huecos Libres` | Calcula slots disponibles en Google Calendar |
| `Gemini IA - Cerebro Central` | Clasifica intención del mensaje |
| `Motor Estado Multi-turno` | Núcleo de la máquina de estados |
| `Router Accion Motor` | Enruta según acción (AGENDAR, ESCALAR, RESPONDER, CANCELAR) |
| `Preparar Respuesta Final` | Unifica mensaje y contexto final |
| `Guardar Contexto Cache` | Persiste estado en memoria (n8n vars) |
| `Guardar Contexto Sheets` | Persiste estado en Google Sheets |
| `Enviar Email Staff Gmail` | Notifica al staff sobre citas y escalados |
| `GHL Upsert Contact` | Sincroniza contacto con GoHighLevel |

---

## 🛡️ Seguridad

- Todas las API keys y tokens deben almacenarse como **variables de entorno** — nunca hardcodeados
- El archivo `SECURITY.md` detalla qué no debe ser commitado (tokens, pinData, datos reales)
- El endpoint de verificación del webhook gestiona la autenticación challenge-response de Meta
- Rate limiting por conversación para prevenir abusos
- Filtrado de mensajes vacíos y eventos de "read" para evitar bucles

---

## 📄 Licencia

Licencia MIT — libre para usar, modificar y distribuir con atribución.

---

## 🤝 Contribuciones

Los pull requests son bienvenidos. Para cambios importantes, abre primero un issue para discutir lo que te gustaría cambiar.

---

## 👤 Autor

**Alejandro Peralta** — Especialista en Automatización de Procesos

- GitHub: [@alejandro-orbis](https://github.com/alejandro-orbis)
- LinkedIn: [linkedin.com/in/alejandro-orbis](https://linkedin.com/in/alejandro-orbis)
- Email: [alejandro@orbisautomations.com](mailto:alejandro@orbisautomations.com)

---

*Construido con ❤️ usando [n8n](https://n8n.io), [Google Gemini](https://deepmind.google/technologies/gemini/) y la Meta Business API.*

*Creado para eliminar el trabajo operativo repetitivo de gestión de citas — para que las clínicas se enfoquen en lo que realmente importa: sus pacientes.*
