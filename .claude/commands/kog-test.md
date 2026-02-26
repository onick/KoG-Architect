# /kog-test — Analisis de Cobertura de Tests

Eres el **KOG Monorepo Principal Architect**. Analiza la cobertura de tests del monorepo y detecta gaps criticos.

## Instrucciones

### Paso 1: Escanear Tests Existentes

Para cada servicio en `services/` y app en `apps/`:

1. Cuenta archivos `.ts` totales (excluyendo `.spec.ts`, `.d.ts`, `index.ts`).
2. Cuenta archivos `.spec.ts` existentes.
3. Verifica que cada `.spec.ts` tiene al menos 1 test case (`it()` o `test()`).
4. Identifica archivos criticos SIN test:
   - `.service.ts` sin `.service.spec.ts` = 🔴
   - `.controller.ts` sin `.controller.spec.ts` = 🟡
   - `.gateway.ts` sin `.gateway.spec.ts` = 🔴
   - `.guard.ts` sin `.guard.spec.ts` = 🟡
   - `.processor.ts` (Bull) sin `.processor.spec.ts` = 🔴

### Paso 2: Clasificar por Criticidad

```
CRITICAL (must have tests in Phase 1):
  - game-engine/*.service.ts
  - game-engine/*.gateway.ts
  - clock-service/*.service.ts
  - websocket-gateway/*.gateway.ts

HIGH (must have tests in Phase 2):
  - anticheat-service/*.processor.ts
  - anticheat-service/*.service.ts
  - matchmaking/*.service.ts
  - rating-service/*.service.ts

MEDIUM (Phase 3-4):
  - api-gateway/*.guard.ts
  - api-gateway/*.controller.ts
  - analytics/*.service.ts
```

### Paso 3: Generar Reporte

```
═══════════════════════════════════════════
  KOG MONOREPO — Test Coverage Analysis
  Date: YYYY-MM-DD
═══════════════════════════════════════════

📊 GLOBAL COVERAGE
  Files: [X] total | [Y] with tests | [Z%] coverage
  Target: 70% global, 80% critical services

🔴 CRITICAL GAPS (no tests, high risk)
  services/game-engine/src/game-engine.service.ts
  services/game-engine/src/move-validation.service.ts
  services/clock-service/src/clock.service.ts
  ...

🟡 HIGH GAPS (no tests, medium risk)
  services/anticheat-service/src/stockfish.processor.ts
  ...

📦 PER-SERVICE BREAKDOWN
  game-engine:      ██░░░░░░░░ 16%  (2/12 files)  [🔴 Phase 1 target: 80%]
  clock-service:    ░░░░░░░░░░  0%  (0/8 files)   [🔴 Phase 1 target: 80%]
  websocket-gw:     █░░░░░░░░░ 10%  (1/10 files)  [🟡 Phase 3 target: 60%]
  anticheat:        ██░░░░░░░░ 20%  (2/10 files)  [🟡 Phase 2 target: 70%]
  matchmaking:      ███░░░░░░░ 30%  (3/10 files)  [🟡 Phase 2 target: 70%]
  rating-service:   █░░░░░░░░░ 15%  (1/7 files)   [🟡 Phase 2 target: 70%]
  analytics:        ░░░░░░░░░░  0%  (0/5 files)   [🟢 Phase 3 target: 50%]
  api-gateway:      ░░░░░░░░░░  5%  (0/6 files)   [🟢 Phase 4 target: 50%]

🎯 PRIORITY TEST WRITING ORDER
  1. game-engine.service.spec.ts — Move validation (CRITICAL)
  2. clock.service.spec.ts — Time control (CRITICAL)
  3. game.gateway.spec.ts — WebSocket events (CRITICAL)
  4. stockfish.processor.spec.ts — Queue worker (HIGH)
  ...

📝 DELEGATION SUGGESTIONS
  - /kog-delegate game-engine "write .spec.ts for MoveValidationService"
  - /kog-delegate clock-service "write .spec.ts for ClockService"
```

### Paso 4: Actualizar Estado

Actualiza `.kog-architect/test-coverage.md` con los numeros reales detectados.

### Reglas
- Coverage se calcula por ARCHIVOS con test, no por lineas (no tenemos jest --coverage aqui).
- Un .spec.ts vacio (0 test cases) NO cuenta como cubierto.
- Prioriza tests de servicios CRITICAL sobre los demas.
- Sugiere delegaciones concretas con `/kog-delegate` para los gaps mas criticos.
