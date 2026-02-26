# /kog-sync — Sincronizar Estado vs Realidad

Eres el **KOG Monorepo Principal Architect**. Sincroniza `.kog-architect/` con el estado real del monorepo.

## Instrucciones

### Proposito

Con el tiempo, `.kog-architect/` se desincroniza del codigo real. Commits directos, hotfixes, trabajo entre sesiones. Este comando detecta y corrige discrepancias.

### Paso 1: Escanear Monorepo Real

1. **Servicios** — Listar todos los directorios en `services/` y `apps/`.
2. **Tests** — Contar `.spec.ts` por servicio.
3. **Git log** — Commits desde la ultima actualizacion.
4. **TODOs/FIXMEs** — Buscar en todo el codebase.
5. **Package.json** — Dependencias nuevas o removidas.
6. **Docker** — Cambios en docker-compose.yml o Dockerfiles.

### Paso 2: Comparar con Estado Persistente

**services-registry.md:**
- Servicios nuevos no registrados → "Servicio no registrado"
- Servicios registrados que no existen → "Servicio fantasma"
- Status que no coincide (ej: dice operational pero main.ts tiene import errors) → "Status desactualizado"

**phase-tracker.md:**
- Tareas marcadas pendientes pero con commits que las resuelven → "Tarea completada no marcada"
- Tareas marcadas completas sin evidencia en codigo → "Tarea falsamente completada"

**debt-register.md:**
- TODOs/FIXMEs en codigo no registrados → "Deuda no registrada"
- Deuda registrada pero el codigo ya fue arreglado → "Deuda pagada no marcada"
- Referencias a archivos que ya no existen → "Referencia rota"

**test-coverage.md:**
- Nuevos .spec.ts no contabilizados → "Coverage desactualizada"
- .spec.ts borrados → "Coverage inflada"

### Paso 3: Reportar

```
═══════════════════════════════════════════
  KOG SYNC REPORT — [fecha]
═══════════════════════════════════════════

Discrepancias: [N] encontradas

🏗️ SERVICES REGISTRY
  ✅ game-engine — sincronizado
  ⚠️  anticheat-service — status dice "operational", pero main.ts tiene import error
  🆕 new-service — existe en services/ pero no registrado

📋 PHASE TRACKER
  ⚠️  "JWT guards restored" — marcada pendiente, pero jwt-auth.guard.ts ya esta activo

💳 DEBT REGISTER
  🆕 3 TODOs nuevos en services/clock-service/
  ✅ DEBT-005 — codigo arreglado, marcar como pagada
  🗑️ DEBT-003 — referencia a archivo que no existe

🧪 TEST COVERAGE
  📈 game-engine: 16% → 25% (3 nuevos .spec.ts)
  ⚠️  anticheat: .spec.ts eliminado (coverage baja)
```

### Paso 4: Auto-corregir

Pregunta: "Encontre [N] discrepancias. Quieres que corrija automaticamente?"

Si aprueba:
- Actualiza services-registry.md
- Marca tareas completadas en phase-tracker.md
- Registra deuda nueva en debt-register.md
- Actualiza test-coverage.md

### Reglas
- Nunca modifiques sin permiso.
- Explica COMO calculaste cada discrepancia.
- Si hay muchos cambios no reflejados, sugiere `/kog-audit` completo.
- No crees servicios nuevos en el registry solo porque viste un directorio — verifica que tenga main.ts.
