# 🏰 KOG Monorepo Principal Architect

**Claude Code Plugin** — Principal Staff Engineer + Real-Time Systems Architect para monorepos de juegos de ajedrez en tiempo real (NestJS + Next.js + Redis/Socket.IO).

## Qué hace

Orquesta, estabiliza y escala un ecosistema de microservicios enfocado en gameplay de alta concurrencia, motores de ajedrez y sistemas anti-trampas. Mantiene estado persistente en `.kog-architect/`.

### Tres sombreros

- **Real-Time Systems Architect** — Sub-100ms latency, horizontal scaling, WebSocket resilience
- **Security & Anti-Cheat Engineer** — Authoritative backend, JWT guards, Stockfish/Lc0 pipelines
- **Monorepo Discipline Officer** — Shared types, consistent testing, clean module boundaries

## Installation

```
/plugin marketplace add onick/KoG-Architect
```

Then run `/plugin menu` to install. Restart Claude Code after.

## Commands

| Command | Purpose |
|---------|---------|
| `/kog-init` | Bootstrap: scan monorepo, generate `.kog-architect/` state |
| `/kog-status` | Dashboard: services health, phase progress, debt summary |
| `/kog-phase` | Manage phases: view current, advance, or plan next |
| `/kog-audit` | Full architecture audit (7 dimensions, score /35) |
| `/kog-delegate` | Generate precise agent delegation for a task |
| `/kog-test` | Test coverage analysis and gap detection |
| `/kog-incident` | Log, analyze, and post-mortem production incidents |
| `/kog-sync` | Sync `.kog-architect/` vs actual codebase |

## 5 Specialized Agents

| Agent | Domain |
|-------|--------|
| **nestjs-backend** | NestJS modules, DI, DTOs, Prisma, health checks |
| **realtime-game** | game-engine, clock-service, websocket-gateway, Redis atomics |
| **anticheat-queues** | Stockfish/Lc0 workers, Bull queues, circuit breakers |
| **nextjs-frontend** | Next.js 15 App Router, Socket.IO client, shared types |
| **devops-sre** | Docker, K8s, GitHub Actions, probes, monitoring |

## 4-Phase Stabilization Roadmap

| Phase | Goal | Exit Criteria |
|-------|------|--------------|
| 1. Core Coverage & Security Seal 🔒 | Prevent production collapse | 80% test coverage on critical services, zero auth bypass |
| 2. Fix Broken Architecture 🔧 | Repair disconnected data flows | All services boot, queues have retry + DLQ |
| 3. Real-Time Optimization ⚡ | Support 5K+ concurrent connections | Reconnection works, clock drift < 50ms |
| 4. Frontend Sync & API Gateway 🌐 | Efficient client-server communication | Zero direct backend calls, rate limiting active |

## Non-Negotiable Guardrails

1. **Backend is Authoritative** — Frontend is a view. Server validates all moves with chess.js
2. **Database Polyglot** — CockroachDB (relational), MongoDB (documents), Redis (active state)
3. **Async by Default** — Bull Queues for CPU-heavy work, never block the event loop
4. **Minimal WebSocket Payloads** — Deltas only, full state on reconnect
5. **No Feature Without Tests** — Every new file gets a `.spec.ts`
6. **Circuit Breakers** — Cross-service deps have fallbacks
7. **Single Source of Truth for Types** — Shared packages, never duplicate

## Persistent State

```
.kog-architect/
├── services-registry.md      # Service health, coverage, dependencies
├── phase-tracker.md           # Current phase with checkboxes
├── debt-register.md           # Tech debt with severity and deadlines
├── test-coverage.md           # Per-service test tracking
├── incident-log.md            # Production incidents and post-mortems
└── architecture-decisions.md  # ADRs (Architecture Decision Records)
```

## Stack

- **Runtime:** Node.js >= 18, TypeScript strict
- **Backend:** NestJS (modular microservices)
- **Frontend:** Next.js 15, React 18, Tailwind
- **Real-Time:** Socket.IO + Redis Adapter
- **Databases:** Redis, CockroachDB (Prisma), MongoDB
- **Queues:** Bull (Redis-backed)
- **Chess:** chess.js, Stockfish, Lc0
- **Infra:** Docker, Kubernetes, GitHub Actions

## Structure

```
KoG-Architect/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── .claude/
│   ├── commands/
│   │   ├── kog-init.md
│   │   ├── kog-status.md
│   │   ├── kog-phase.md
│   │   ├── kog-audit.md
│   │   ├── kog-delegate.md
│   │   ├── kog-test.md
│   │   ├── kog-incident.md
│   │   └── kog-sync.md
│   └── skills/
│       └── kog-monorepo-architect/
│           ├── SKILL.md
│           ├── agents/
│           │   ├── nestjs-backend-agent.md
│           │   ├── realtime-game-agent.md
│           │   ├── anticheat-queues-agent.md
│           │   ├── nextjs-frontend-agent.md
│           │   └── devops-sre-agent.md
│           └── references/
│               ├── nestjs-microservices.md
│               ├── realtime-architecture.md
│               ├── chess-engine-integration.md
│               ├── bull-queues-patterns.md
│               ├── testing-strategy.md
│               ├── security-anticheat.md
│               ├── database-polyglot.md
│               └── devops-k8s.md
├── LICENSE
└── README.md
```

## License

MIT

---

Creado por [Contan2](https://contan2.com)
