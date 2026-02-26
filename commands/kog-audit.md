---
name: kog-audit
description: Auditoria completa de arquitectura en 7 dimensiones (seguridad, resiliencia, testing, performance, arquitectura, devops, documentacion).
---

# /kog-audit — Auditoria Completa de Arquitectura

Eres el **KOG Monorepo Principal Architect**. Ejecuta una auditoria exhaustiva del ecosistema.

## Instrucciones

Analiza estas 7 dimensiones y genera un reporte con score por dimension.

### 1. Seguridad (Auth & Anti-Cheat)
- JWT guards activos en TODOS los endpoints y gateways?
- simple-auth eliminado?
- WebSocket connections requieren auth?
- Rate limiting configurado en api-gateway?
- Anticheat pipeline funcional?
- Secrets fuera del codigo (no hardcoded)?
Score: [1-5]

### 2. Resiliencia (Circuit Breakers & Fallbacks)
- Circuit breakers en llamadas cross-service?
- Bull queues con retry + backoff + DLQ?
- Si anticheat cae, game-engine sigue?
- Si analytics cae, nada se rompe?
- Reconnection WebSocket maneja todos los edge cases?
- Health probes configurados?
Score: [1-5]

### 3. Testing
- Cada servicio tiene .spec.ts?
- Coverage global vs target?
- Tests cubren happy path Y error paths?
- Tests de integracion con Redis real?
- WebSocket tests para conexion/desconexion?
- CI ejecuta tests antes de build?
Score: [1-5]

### 4. Performance (Real-Time)
- Move processing < 50ms?
- WebSocket payloads son deltas (no full state)?
- Redis operations son atomicas (MULTI/EXEC)?
- Event loop no bloqueado por chess.js analysis?
- Clock drift < 50ms con lag compensation?
- Redis Adapter para multi-instance?
Score: [1-5]

### 5. Arquitectura (Module Boundaries)
- Comunicacion inter-service via Redis/Bull (no HTTP directo)?
- Tipos compartidos via packages/shared-types?
- No hay duplicacion de DTOs?
- Cada servicio tiene responsabilidad unica?
- Database boundaries respetados (Redis/CockroachDB/MongoDB)?
Score: [1-5]

### 6. DevOps (Infra & CI/CD)
- Dockerfiles con multi-stage builds?
- docker-compose funcional para dev local?
- GitHub Actions con tests + build?
- K8s manifests con resource limits?
- Monitoring configurado?
Score: [1-5]

### 7. Documentacion (State & ADRs)
- .kog-architect/ actualizado?
- services-registry.md refleja realidad?
- ADRs documentados para decisiones clave?
- README por servicio?
Score: [1-5]

### Formato del Reporte

```
===============================================
  KOG MONOREPO — Architecture Audit
  Date: YYYY-MM-DD
===============================================

SCORE GENERAL: [X/35]

Seguridad:     [X/5]
Resiliencia:   [X/5]
Testing:       [X/5]
Performance:   [X/5]
Arquitectura:  [X/5]
DevOps:        [X/5]
Documentacion: [X/5]

TOP 3 PROBLEMAS CRITICOS
1. [Problema + servicio + accion]
2. [Problema + servicio + accion]
3. [Problema + servicio + accion]

TOP 3 FORTALEZAS
1. [Que funciona bien]
2. [Que funciona bien]
3. [Que funciona bien]

ACCIONES INMEDIATAS (esta semana)
- [ ] [Accion] -> Agente: [nestjs|realtime|anticheat|nextjs|devops]
- [ ] [Accion] -> Agente: [...]
- [ ] [Accion] -> Agente: [...]
```

### Reglas
- Score 30-35 = excelente, 22-29 = saludable, 15-21 = necesita atencion, <15 = emergencia.
- Cada accion inmediata debe especificar QUE agente la ejecuta.
- Se brutalmente honesto. Un audit suavizado es inutil.
