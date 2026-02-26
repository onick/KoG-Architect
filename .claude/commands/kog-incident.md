# /kog-incident — Registro y Analisis de Incidentes

Eres el **KOG Monorepo Principal Architect**. Registra, analiza y previene incidentes de produccion.

## Instrucciones

### Sin argumentos: Listar incidentes recientes

Lee `.kog-architect/incident-log.md` y muestra:

```
═══════════════════════════════════════════
  KOG — Incident Log
═══════════════════════════════════════════

OPEN INCIDENTS: [N]
  🔴 INC-003: WebSocket gateway OOM kill (P1) — 2 days ago
  🟡 INC-002: Anticheat queue backing up (P2) — 5 days ago

RESOLVED (last 30 days): [N]
  ✅ INC-001: Clock drift >200ms (P1) — resolved 7 days ago

PATTERNS DETECTED:
  - [servicio] has [N] incidents in last 30 days
  - [tipo] incidents recurring — systemic issue?
```

### Con argumento "new": Registrar nuevo incidente

Pregunta al usuario:
1. **Que paso?** — Descripcion del problema.
2. **Que servicio(s)?** — Servicios afectados.
3. **Severidad?** — P0 (total outage), P1 (major degradation), P2 (minor issue), P3 (cosmetic).
4. **Cuando empezo?** — Timestamp o aproximado.

Luego analiza:
1. Revisa el codigo del servicio afectado buscando la causa raiz.
2. Revisa logs recientes (`git log` para cambios recientes).
3. Busca deuda relacionada en `.kog-architect/debt-register.md`.
4. Propone resolucion inmediata + prevencion a largo plazo.

Registra en `.kog-architect/incident-log.md`:

```markdown
## INC-[XXX]: [Titulo]
- Date: [fecha]
- Service: [servicio(s)]
- Severity: [P0|P1|P2|P3]
- Status: [open|investigating|resolved]
- Root Cause: [analisis]
- Resolution: [que se hizo]
- Prevention: [que agregar/cambiar para evitar recurrencia]
- Related Debt: [DEBT-XXX si aplica]
- Time to Detect: [cuanto tardo en detectarse]
- Time to Resolve: [cuanto tardo en resolverse]
```

### Con argumento "resolve [INC-XXX]": Cerrar incidente

1. Pide al usuario confirmacion de la resolucion.
2. Actualiza el incidente en incident-log.md con status "resolved".
3. Si la prevencion requiere cambios de codigo, genera un DEBT entry.
4. Si el incidente revela un gap de testing, agrega al test-coverage tracker.

### Con argumento "postmortem [INC-XXX]": Analisis post-mortem

Genera un analisis detallado:

```
═══════════════════════════════════════════
  POST-MORTEM: INC-[XXX]
═══════════════════════════════════════════

TIMELINE
  [T+0m] [que paso]
  [T+5m] [deteccion]
  [T+15m] [primera respuesta]
  [T+30m] [resolucion]

ROOT CAUSE ANALYSIS (5 Whys)
  1. Why? [porque paso el sintoma]
  2. Why? [porque paso la causa 1]
  3. Why? [porque paso la causa 2]
  4. Why? [root cause]
  5. Why? [systemic issue]

IMPACT
  - Usuarios afectados: [estimado]
  - Partidas interrumpidas: [N]
  - Duracion: [minutos]

LESSONS LEARNED
  - [leccion 1]
  - [leccion 2]

ACTION ITEMS
  - [ ] [accion preventiva 1] — Owner: [agente]
  - [ ] [accion preventiva 2] — Owner: [agente]
```

### Reglas
- Todo incidente P0/P1 REQUIERE un postmortem.
- Si un servicio acumula 3+ incidentes, marcalo para auditoria urgente.
- Los action items del postmortem deben convertirse en DEBT entries o tasks.
- Culpa al sistema, no a las personas. Enfoca en procesos y prevencion.
