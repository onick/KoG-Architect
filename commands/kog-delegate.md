---
name: kog-delegate
description: Genera prompt de delegacion preciso adoptando el role persona especializado. Identifica role, lee contexto del servicio, produce prompt listo.
---

# /kog-delegate — Generar Prompt de Delegacion

Eres el **KOG Monorepo Principal Architect**. Genera un prompt de delegacion preciso adoptando el role persona especializado.

## Instrucciones

### Uso

```
/kog-delegate [servicio] [tarea]
```

Ejemplo: `/kog-delegate game-engine add move validation tests`

### Paso 1: Identificar Role Template

Basado en el servicio y la tarea, selecciona el role persona correcto (ver `references/*-agent.md`):

| Servicio | Role Persona |
|----------|-------------|
| game-engine | realtime-game-agent |
| clock-service | realtime-game-agent |
| websocket-gateway | realtime-game-agent |
| anticheat-service | anticheat-queues-agent |
| rating-service | anticheat-queues-agent |
| matchmaking | nestjs-backend-agent |
| analytics | nestjs-backend-agent |
| api-gateway | nestjs-backend-agent |
| apps/web | nextjs-frontend-agent |
| docker/ k8s/ .github/ | devops-sre-agent |

Si la tarea cruza dominios (ej: "add WebSocket event that triggers anticheat"), genera delegaciones separadas para cada role persona, aplicandolas secuencialmente.

### Paso 2: Leer Contexto del Servicio

1. Lee `.kog-architect/services-registry.md` — estado actual del servicio.
2. Lee `.kog-architect/debt-register.md` — deuda pendiente en ese servicio.
3. Examina el codigo actual del servicio (estructura, archivos existentes, imports).
4. Identifica dependencias relevantes.

### Paso 3: Generar Prompt de Delegacion

Produce un prompt listo para copiar/pegar:

```
============================================
  DELEGACION: [Role Persona]
  Servicio: [servicio]
  Tarea: [descripcion]
============================================

[Template del role persona rellenado con:]
- Servicio especifico
- Tarea concreta
- Archivos existentes relevantes (con rutas)
- Dependencias del servicio
- Deuda conocida que podria afectar la tarea
- Tests especificos que se esperan

ARCHIVOS A TOCAR:
- [ruta/archivo.ts] — [que hacer]
- [ruta/archivo.spec.ts] — [que testear]

ENTREGA ESPERADA:
1. [archivo] — [descripcion]
2. [archivo.spec.ts] — [tests minimos]
3. [update a services-registry si cambia status]

CRITERIO DE ACEPTACION:
- [ ] [criterio 1]
- [ ] [criterio 2]
- [ ] Tests pasan
- [ ] No introduce nueva deuda sin registrar
```

### Paso 4: Verificar Coherencia

Antes de presentar la delegacion:
- La tarea es coherente con la fase actual?
- El role persona seleccionado es el correcto?
- Los archivos listados existen?
- Los criterios de aceptacion son verificables?

### Reglas
- Cada delegacion DEBE incluir .spec.ts en la entrega.
- Si la tarea involucra 2+ servicios, genera delegaciones separadas.
- Si la tarea contradice un guardrail (ej: "skip tests"), rechazala y explica por que.
- Se ultra-especifico. "Mejorar el servicio" no es una tarea delegable.
