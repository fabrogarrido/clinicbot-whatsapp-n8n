# FAQ Lead Agent — WhatsApp + GPT-4o-mini (Template)

Agente conversacional para atención automática 24/7 vía WhatsApp. Califica leads recopilando los datos clave del cliente (prenda, talle, color) y deriva a una vendedora humana cuando la consulta lo requiere. Desarrollado originalmente para una tienda de indumentaria en Buenos Aires, reconvertido en **template white label** adaptable a cualquier negocio de retail o e-commerce.

**Stack:** n8n (self-hosted) · GPT-4o-mini · Meta WhatsApp Business API · PostgreSQL · Google Sheets · Docker · VPS  
**Estado:** Prototipo funcional validado (PoC)

---

## ¿Qué hace?

1. Recibe mensajes de WhatsApp via Meta Business API
2. Filtra status updates de Meta (delivered/read) para evitar loops infinitos
3. Extrae el número de teléfono como identificador de sesión
4. Procesa el mensaje con GPT-4o-mini usando un system prompt orientado a calificación de leads
5. Mantiene memoria de conversación por cliente en PostgreSQL (últimos 10 mensajes)
6. Cuando tiene los 3 datos del lead (prenda + talle + color), extrae la información estructurada y la guarda en Google Sheets
7. Responde al cliente por WhatsApp con el mensaje generado por el agente

## Flujo de nodos

```
WhatsApp Trigger
      │
      ▼
   IF1 (¿es mensaje real?)  ──── No ──→ No Operation
      │ Sí
      ▼
  Edit Fields (extrae teléfono y mensaje)
      │
      ▼
  AI Agent (GPT-4o-mini + Postgres Memory)
      │
      ▼
  Code in JavaScript (extrae LEAD_DATA si existe)
      │
      ├──→ Send Message (WhatsApp)
      │
      └──→ IF (¿lead completo?)
                │ Sí
                ▼
          Append row in sheet (Google Sheets)
```

## Nodos en detalle

| Nodo | Tipo | Función |
|------|------|---------|
| WhatsApp Trigger | Trigger | Escucha mensajes entrantes via webhook de Meta |
| IF1 | IF | Filtra status updates (delivered/read) — solo procesa mensajes reales con `messages[0].from` |
| Edit Fields | Set | Extrae `telefono` (`contacts[0].wa_id`) y `mensaje` (`messages[0].text.body`) |
| AI Agent | LangChain Agent | Procesa el mensaje con GPT-4o-mini y el system prompt de calificación |
| OpenAI Chat Model | Sub-nodo | GPT-4o-mini — modelo de lenguaje del agente |
| Postgres Chat Memory | Sub-nodo | Memoria persistente por sesión; session key = número de teléfono; context window = 10 mensajes |
| Code in JavaScript | Code | Extrae el bloque `LEAD_DATA:{...}` del output del agente; devuelve mensaje limpio + campos estructurados + flag `guardar_en_sheets` |
| IF | IF | Guarda en Sheets solo cuando `guardar_en_sheets === true` (lead con los 3 datos completos) |
| Send Message | WhatsApp | Envía la respuesta del agente al cliente |
| Append row in sheet | Google Sheets | Guarda el lead calificado: Fecha, Teléfono, Prenda, Talle, Color, Resumen |

## Lógica de extracción de leads

El system prompt instruye al agente a incluir al final de su respuesta un bloque estructurado cuando tiene los 3 datos:

```
LEAD_DATA:{"prenda":"campera","talle":"M","color":"negra","resumen":"cliente busca campera M negra"}
```

El nodo **Code in JavaScript** parsea este bloque con regex, separa el mensaje limpio del dato estructurado, y activa el flag `guardar_en_sheets` solo cuando los 3 campos están completos. Esto evita guardar filas vacías en cada mensaje del cliente.

## Configuración requerida

### 1. Meta WhatsApp Business API
- Crear una app en [Meta for Developers](https://developers.facebook.com/)
- Configurar el webhook apuntando a la URL del trigger de n8n
- Token de verificación: cualquier string que definas (configurable en n8n)
- **Importante:** Crear un System User con token permanente para evitar que el token venza cada pocas horas
  - `business.facebook.com → Configuración del negocio → Usuarios del sistema → Agregar`
  - Rol: Administrador | Caducidad: Nunca | Permisos: `whatsapp_business_messaging` + `whatsapp_business_management`

### 2. Credenciales en n8n
| Credencial | Nodo | Qué configurar |
|-----------|------|----------------|
| WhatsApp OAuth account | WhatsApp Trigger | App ID + App Secret de Meta |
| WhatsApp account | Send Message | Access Token permanente de Meta |
| OpenAI account | OpenAI Chat Model | API Key de OpenAI |
| Postgres account | Postgres Chat Memory | Host, puerto, usuario y contraseña de tu PostgreSQL |
| Google Sheets OAuth2 | Append row in sheet | OAuth2 con Google Cloud Console |

### 3. Google Sheets
- Crear una hoja con las columnas: `Fecha | Telefono | Prenda | Talle | Color | Resumen`
- Reemplazar `TU_GOOGLE_SHEET_ID` en el nodo **Append row in sheet** con el ID real de tu hoja

### 4. Send Message
- En el nodo **Send Message**, reemplazar `TU_PHONE_NUMBER_ID` con el Phone Number ID de tu app de Meta
- El campo `recipientPhoneNumber` se completa dinámicamente desde el flujo — verificar que el formato sea correcto para tu país

### 5. System prompt
- Reemplazar `[NOMBRE_TIENDA]` y `[CIUDAD]` con los datos reales del negocio
- Actualizar locales, horarios, medios de pago y política de envíos
- Los 3 datos a calificar (prenda, talle, color) son configurables según el negocio

## Adaptación a otros rubros

El template está diseñado para indumentaria pero la estructura es reutilizable. Para adaptarlo:

1. **Cambiar los 3 datos de calificación** en el system prompt (ej: para inmobiliaria: zona, presupuesto, tipo de propiedad)
2. **Actualizar el bloque LEAD_DATA** para que refleje los nuevos campos
3. **Actualizar el nodo Code** para extraer los nuevos campos
4. **Actualizar las columnas** del Google Sheets

## Decisiones técnicas

| Decisión | Alternativa descartada | Razón |
|----------|----------------------|-------|
| GPT-4o-mini | GPT-4o | 15x más barato; calidad suficiente para FAQ y calificación de leads |
| PostgreSQL para memoria | Simple Memory (RAM) | Persiste entre reinicios del servidor; cada cliente mantiene su historial |
| Teléfono como session key | ID automático de n8n | Garantiza que todos los mensajes del mismo número van a la misma conversación |
| IF anti-loop al inicio | Sin filtro | Meta envía status updates (delivered/read) que dispararían el flujo en loop |
| Extracción con regex (LEAD_DATA) | Structured output de OpenAI | Más simple de implementar en n8n sin nodos adicionales |

## Pendientes (v2)

- [ ] Notificación automática a vendedora cuando el agente deriva un cliente
- [ ] Scheduler nocturno: acumular derivaciones fuera de horario y notificar al abrir
- [ ] Switch por local: detectar a qué número escribió el cliente y responder desde ese mismo número
- [ ] Botones inline de Telegram en lugar de respuesta de texto libre

---

*Desarrollado por [Fabricio Garrido](https://linkedin.com/in/fabriciogarrido) — [Itera Digital Hub](https://github.com/fabrogarrido)*
