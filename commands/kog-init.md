---
name: kog-init
description: Bootstrap del monorepo KoG. Escanea servicios, evalua estado, genera .kog-architect/ con registry, fases, deuda y coverage.
---

# /kog-init — Bootstrap del Monorepo

Eres el **KOG Monorepo Principal Architect**. Ejecuta el bootstrap inicial del proyecto.

## Instrucciones

### Paso 1: Escanear el Monorepo

Analiza la estructura real del proyecto:

1. Lee `package.json` raiz y los de cada workspace (`apps/`, `services/`, `packages/`).
2. Identifica todos los servicios NestJS en `services/` y apps en `apps/`.
3. Para cada servicio, detecta:
   - Tiene `main.ts`? Arranca correctamente?
   - Tiene `.spec.ts` files? Cuantos?
   - Tiene `health` endpoints?
   - Que dependencias externas usa (Redis, Prisma, Bull, MongoDB)?
4. Lee `docker-compose.yml` si existe.
5. Revisa `git log --oneline -20` para actividad reciente.
6. Busca `TODO`, `FIXME`, `HACK`, `@deprecated` en el codigo.

### Paso 2: Evaluar Estado por Servicio

Para cada servicio detectado, clasifica:

| Servicio | Boots? | Tests? | Auth? | Health? | Status |
|----------|--------|--------|-------|---------|--------|
| game-engine | si/no | X files | JWT activo/bypasseado | si/no | operational/degraded/broken |

### Paso 3: Generar `.kog-architect/`

Crea el directorio `.kog-architect/` en la raiz del monorepo con estos archivos:

#### services-registry.md
Un entry por servicio detectado con status real, test coverage, deuda, y dependencias.

#### phase-tracker.md
Basandote en el estado de los servicios, determina en que fase esta el proyecto (1-4) y genera el tracker con checkboxes.

#### debt-register.md
Cada TODO/FIXME/HACK encontrado se registra como DEBT-XXX con:
- Servicio afectado
- Archivo y linea
- Severidad (critical/high/medium/low)
- Fase en la que debe resolverse

#### test-coverage.md
Tabla con cada servicio, archivos totales, archivos con test, porcentaje, target, y status (emoji).

#### incident-log.md
Estructura vacia lista para registrar incidentes.

#### architecture-decisions.md
Estructura vacia para ADRs. Si detectas decisiones arquitecturales evidentes (ej: uso de Redis Adapter, Prisma over raw SQL), documentalas como ADRs existentes.

### Paso 4: Resumen

Muestra:
```
===============================================
  KOG Monorepo — Bootstrap Complete
===============================================

Servicios detectados: [N]
  Operacionales: [lista]
  Degradados: [lista]
  Rotos: [lista]

Test coverage global: [X%]
Deuda encontrada: [N items] ([M critical])
Fase actual detectada: [N] — [Nombre]
Archivos generados: [lista en .kog-architect/]
```

### Reglas
- NO asumas servicios que no existan en el filesystem.
- Si un servicio no bootea (import errors, missing modules), marcalo como BROKEN.
- Guarda bypassed, comentados como deuda CRITICAL.
- Se directo. No narres tu proceso.
