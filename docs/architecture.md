# Arquitectura del Sistema: Lead Qualification Multiagent

## Diagrama de Flujo

```mermaid
flowchart TD
    A([Lead entrante - Webhook POST]) --> B[Respuesta HTTP 200 Inmediata]
    A --> C[Data Sanitization - Anti-PII + UUID]
    C --> D[Agente A: Analista - gpt-4o-mini]
    D --> E{Validar Schema Agente A}
    E -->|JSON válido| F[Agente B: Decisor - gpt-4o-mini]
    E -->|Error| ERR1[Log de Error + Alerta]
    F --> G{Circuit Breaker - Confianza + Costo}
    G -->|OK| H[Observability Log]
    G -->|Fallo| ERR2[Abort + Alerta]
    H --> I[(Google Sheets - Telemetría)]
    H --> J{Clasificación}
    J -->|HOT| K[Slack ventas]
    J -->|WARM| L[Email nurturing]
    J -->|COLD| M[Archivo]
    J --> N{Confianza menor 50%?}
    N -->|Sí| O[Slack ops - humano]
```

## Patrón: Handoff (Cadena)

| Agente | Rol | Output |
|--------|-----|--------|
| Agente A (Analista) | Extrae y valida datos | JSON con score de completitud |
| Agente B (Decisor) | Clasifica y decide acción | Hot/Warm/Cold + acción |

## Capas de Seguridad

- Data Sanitization elimina PII antes de llamar a OpenAI
- Validador de schema entre agentes
- Circuit Breaker: corta si costo mayor $0.50 o confianza menor 50%
- Human-in-the-Loop automático

## Variables de Entorno

Ver `.env.example` en la raíz del repositorio.
