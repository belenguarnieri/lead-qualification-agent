# Lead Qualification Agent — Sistema Multiagente con n8n

Sistema de calificación automática de leads comerciales basado en dos agentes de IA especializados. Implementa patrones de orquestación, observabilidad completa y gobernanza de seguridad production-ready.

## Problema que resuelve

Los equipos de ventas reciben leads de múltiples fuentes (formularios web, LinkedIn, eventos) y clasificarlos manualmente consume tiempo valioso. Este sistema automatiza la clasificación en **Hot / Warm / Cold** con trazabilidad completa y escalado automático a humano cuando la confianza de la IA es baja.

**Impacto estimado:** ~8 minutos por lead clasificado manualmente → ~15 segundos con el sistema. Con 50 leads/día: ahorro de 6.25 horas/día del equipo comercial.

---

## Arquitectura

Ver `docs/architecture.md` para el diagrama completo. Resumen del flujo:

```
Webhook → Data Sanitization → Agente A (Analista) → Validación Schema
→ Agente B (Decisor) → Circuit Breaker → Observability Log
→ Google Sheets (telemetría) + Slack (alertas)
```

**Patrón:** Handoff (cadena) — cada agente tiene responsabilidad única y contexto limpio.

---

## Setup

### Prerequisitos

- Cuenta en [n8n.io](https://n8n.io) (cloud) o instancia self-hosted con Docker
- API Key de OpenAI
- Google Sheets configurado con OAuth en n8n
- Bot de Slack configurado con permisos `chat:write`

### 1. Configurar variables de entorno

```bash
cp .env.example .env
# Editar .env con tus credenciales reales
```

### 2. Importar el workflow en n8n

1. Abrir n8n → menú hamburguesa → **Import from file**
2. Seleccionar `workflow_lead_qualification.json`
3. Configurar las credenciales en cada nodo (OpenAI, Google Sheets, Slack)

### 3. Configurar Google Sheets de Observabilidad

Crear una hoja llamada `Logs` con estas columnas en la fila 1:

```
timestamp | transaction_id | company | classification | lead_score |
recommended_action | confidence_score | total_tokens | estimated_cost_usd |
requires_human_review | status
```

### 4. Activar el workflow

Hacer clic en **Activate** en n8n. El webhook estará disponible en:
```
POST https://tu-instancia.n8n.io/webhook/new-lead
```

### 5. Probar con curl

```bash
curl -X POST https://tu-instancia.n8n.io/webhook/new-lead \
  -H "Content-Type: application/json" \
  -d '{
    "company_name": "TechCorp SA",
    "industry": "SaaS",
    "company_size": "50-200 empleados",
    "message": "Necesitamos automatizar nuestro proceso de facturación. Tenemos presupuesto aprobado para Q1.",
    "budget_range": "10000-50000 USD",
    "email": "cto@techcorp.com",
    "source": "LinkedIn"
  }'
```

**Respuesta esperada:**
```json
{ "status": "received", "message": "Lead recibido. Procesando." }
```

---

## Runbook de Incidentes

### ¿Qué hacer si el Agente A falla?

**Síntoma:** El nodo "Validar Schema Agente A" registra `validation_passed: false` o el flujo se detiene en ese nodo.

**Diagnóstico:**
1. Revisar el log en Google Sheets — buscar el `transaction_id` afectado
2. En n8n → Executions → abrir la ejecución fallida → ver el output del nodo "Agente A: Analista de Lead"
3. Verificar si el raw output de OpenAI contiene texto antes o después del JSON (el agente no siguió la instrucción de formato)

**Acciones:**
- Si es error de formato puntual: el flujo captura el error y continúa con datos de fallback. No requiere acción.
- Si falla persistentemente (>3 ejecuciones seguidas): revisar si la API de OpenAI tiene incidentes en [status.openai.com](https://status.openai.com)
- Si el error es `rate_limit_exceeded`: añadir un nodo **Wait** de 60 segundos antes del Agente A

### ¿Qué hacer si el Agente B falla?

**Síntoma:** Circuit Breaker lanza error o el nodo "Validar Schema Agente B" falla.

**Diagnóstico:**
1. Si el error dice `COST_CIRCUIT_BREAKER`: la ejecución consumió tokens en exceso. Revisar si el output del Agente A era inusualmente largo.
2. Si la confianza es < 50%: el lead llegó a Slack #ops automáticamente para revisión manual — no hay acción de urgencia.

**Acciones:**
- Revisar la ejecución en n8n con el `transaction_id` del alerta de Slack
- Para el circuit breaker de costo: revisar si hay loops o prompts que se expandieron

### ¿Qué hacer si Google Sheets no recibe logs?

**Síntoma:** El flujo termina exitosamente pero no aparecen filas en el Sheet.

**Acciones:**
1. Verificar que `OBSERVABILITY_SHEET_ID` en las variables del nodo sea el ID correcto de tu hoja
2. Verificar que la hoja se llama exactamente `Logs` (sensible a mayúsculas)
3. Reconectar las credenciales de Google Sheets en n8n (pueden expirar el token OAuth)

### ¿Qué hacer si Slack no recibe alertas?

1. Verificar que el bot esté invitado al canal: `/invite @nombre-del-bot` en Slack
2. Verificar que los nombres de canal en `.env` incluyen el `#` al inicio
3. Verificar en n8n → Credentials que el token de Slack no expiró

---

## Seguridad

- **Ninguna credencial** está hardcodeada en el workflow — todas se referencian desde las credenciales de n8n o variables de entorno
- El nodo **Data Sanitization** elimina PII (emails completos, teléfonos) antes de cualquier llamada a OpenAI
- **Circuit Breaker** corta ejecuciones que superen el umbral de costo configurado
- **Human-in-the-Loop**: si la confianza del Agente B < 50%, el sistema escala a revisión humana en lugar de actuar autónomamente

---

## Estructura del Repositorio

```
lead-qualification-agent/
├── workflow_lead_qualification.json   # Flujo exportado de n8n (importar directamente)
├── .env.example                       # Plantilla de variables de entorno
├── README.md                          # Este archivo
└── docs/
    └── architecture.md               # Diagrama de arquitectura y decisiones técnicas
```

---

## Criterios de Evaluación Cubiertos

| Criterio | Implementación |
|----------|---------------|
| Orquestación multiagente | Patrón Handoff con Agente A (Analista) y Agente B (Decisor) |
| Robustez ante fallo de API | Nodos de validación capturan errores; el flujo no se rompe si OpenAI falla |
| Seguridad de credenciales | Variables de entorno + nodo Data Sanitization Anti-PII |
| Trazabilidad por ID | `transaction_id` UUID generado al inicio, propagado a todos los logs |
| Observabilidad | Log estructurado en Google Sheets con timestamp, tokens, costo y status |
| Circuit Breaker | Corta ejecución si costo > $0.50 por ejecución o confianza < 50% |
