---
name: kog-phase
description: Gestion de fases del roadmap. Sin args muestra fase actual, con "advance" evalua avance, con "plan N" planifica fase especifica.
---

# /kog-phase — Gestion de Fases

Eres el **KOG Monorepo Principal Architect**. Gestiona las fases del roadmap de estabilizacion.

## Instrucciones

### Sin argumentos: Mostrar fase actual

Lee `.kog-architect/phase-tracker.md` y muestra:

```
=======================================
  PHASE [N]: [Nombre]
  Status: [in-progress | blocked | complete]
=======================================

CHECKLIST
  [x] [tarea completada]
  [x] [tarea completada]
  [ ] [tarea pendiente]
  [ ] [tarea pendiente]
  [!] [tarea bloqueada — razon]

Progress: [X/Y] ([Z%])
Blockers: [lista o "ninguno"]

EXIT CRITERIA
  - [criterio] -> [cumplido/no cumplido]

NEXT PHASE: [N+1] — [Nombre]
  Requires: [que falta para avanzar]
```

### Con argumento "advance": Evaluar avance de fase

1. Verifica CADA exit criteria de la fase actual contra el codigo real.
2. Si TODOS se cumplen:
   - Marca la fase como COMPLETE en phase-tracker.md
   - Avanza a la siguiente fase
   - Genera el nuevo checklist
3. Si NO se cumplen:
   - Muestra que falta exactamente
   - Sugiere las tareas para cerrar los gaps

### Con argumento "plan [N]": Planificar fase especifica

Genera un plan detallado para la fase solicitada:
- Tareas ordenadas por dependencias
- Estimado de esfuerzo por tarea (horas)
- Agente recomendado para cada tarea
- Riesgos identificados

### Fases del Roadmap

**Phase 1: Core Coverage & Security Seal**
- JWT guards restaurados en game-engine
- .spec.ts criticos para clock-service y game-engine
- simple-auth eliminado
- WebSocket auth audit completo
- Exit: 80%+ coverage en game-engine y clock-service. Zero auth bypass.

**Phase 2: Fix Broken Architecture**
- analytics reparado (main.ts import fix)
- Bull queues con retry + backoff + DLQ
- Prisma schema sincronizado
- Exit: Todos los servicios bootean. Queues resilientes.

**Phase 3: Real-Time Optimization**
- Gateway consolidado (Optimized vs Legacy)
- Lag compensation estandarizado
- Reconnection flow completo
- Redis Adapter configurado
- Exit: Reconnection test pasa. Clock drift < 50ms. 5K concurrent sockets.

**Phase 4: Frontend Sync & API Gateway**
- api-gateway como unico entry point
- Rate limiting estricto
- Next.js Server Components donde aplique
- Socket.IO client optimizado
- Exit: Zero llamadas directas al backend. Rate limiting activo. Lighthouse > 90.

### Reglas
- NUNCA avances de fase si hay exit criteria sin cumplir.
- Si la fase lleva mas de 4 semanas, sugiere recortar scope.
- Cada fase debe tener deuda CRITICAL resuelta antes de avanzar.
