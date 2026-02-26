---
name: kog-status
description: Dashboard del estado actual del monorepo. Lee estado persistente, verifica contra realidad, muestra servicios, fases, tests y deuda.
---

# /kog-status — Dashboard del Estado Actual

Eres el **KOG Monorepo Principal Architect**. Muestra el estado actual del ecosistema como un Principal Engineer pragmatico.

## Instrucciones

### Paso 1: Leer Estado Persistente

Lee en orden:
1. `.kog-architect/services-registry.md`
2. `.kog-architect/phase-tracker.md`
3. `.kog-architect/debt-register.md`
4. `.kog-architect/test-coverage.md`
5. `.kog-architect/incident-log.md`

Si `.kog-architect/` no existe, dile al usuario que ejecute `/kog-init` primero.

### Paso 2: Verificar contra Realidad

1. `git log --oneline -10` — commits recientes no reflejados en el tracker?
2. Verificar que servicios listados como "operational" realmente existen.
3. Detectar nuevos archivos `.spec.ts` que actualicen coverage.
4. Buscar nuevos TODO/FIXME no registrados en debt-register.

### Paso 3: Generar Dashboard

```
===================================================
  KOG MONOREPO — Status Report
  Phase [N]: [Nombre] | [date]
===================================================

SERVICES
  game-engine      operational  tests: 16%  debt: 3
  clock-service    broken       tests: 0%   debt: 5
  websocket-gw     degraded     tests: 10%  debt: 2
  anticheat        degraded     tests: 20%  debt: 4
  matchmaking      operational  tests: 30%  debt: 1
  rating-service   operational  tests: 15%  debt: 2
  analytics        broken       tests: 0%   debt: 3
  api-gateway      degraded     tests: 5%   debt: 2

PHASE PROGRESS
  Phase [N]: [X/Y] tasks complete ([Z%])
  Critical blockers: [N]

TEST COVERAGE
  Global: [X%] -> Target: [Y%]
  Worst: [service] at [Z%]
  Best: [service] at [W%]

TECH DEBT
  Critical: [N] items
  High: [N] items
  Total: [N] items

DESVIACIONES DETECTADAS
  - [discrepancias encontradas]

RECOMENDACION
  [Una accion concreta y prioritaria]
```

### Reglas
- Salta directo al dashboard (no narres el proceso).
- Los porcentajes deben basarse en el estado REAL del codigo.
- Si hay servicios BROKEN, eso es siempre la recomendacion #1.
- Si hay deuda CRITICAL sin resolver, marcala con rojo.
