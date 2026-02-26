# NestJS Microservices Patterns for KOG Monorepo

## Module Architecture

Every service in the monorepo follows this structure:

```
services/[service-name]/
├── src/
│   ├── [service].module.ts        # Root module
│   ├── [service].controller.ts    # HTTP endpoints (if any)
│   ├── [service].service.ts       # Core business logic
│   ├── [service].gateway.ts       # WebSocket gateway (if real-time)
│   ├── [service].spec.ts          # Tests (MANDATORY)
│   ├── dto/
│   │   ├── [action].dto.ts        # Input validation
│   │   └── [response].dto.ts      # Output shapes
│   ├── guards/
│   │   └── jwt-auth.guard.ts      # Auth enforcement
│   ├── interfaces/
│   │   └── [service].interfaces.ts
│   └── providers/
│       └── [external].provider.ts  # Redis, Bull, etc.
├── test/
│   └── [service].e2e-spec.ts
└── tsconfig.json
```

## Dependency Injection Rules

1. **Services inject services, never modules.**
   ```typescript
   // GOOD
   constructor(private readonly clockService: ClockService) {}

   // BAD - importing the module directly
   import { ClockModule } from '../clock/clock.module';
   ```

2. **Cross-service communication goes through Redis Pub/Sub or Bull Queues.**
   ```typescript
   // GOOD - async message
   await this.redisClient.publish('game:move', JSON.stringify(moveData));

   // BAD - direct HTTP call between services
   await this.httpService.post('http://clock-service/api/tick', data);
   ```

3. **Shared providers are registered in a `CoreModule` imported by all services.**

## DTO Validation Pattern

Every endpoint and WebSocket event MUST validate input:

```typescript
import { IsString, IsInt, Min, Max, IsUUID } from 'class-validator';

export class MakeMoveDto {
  @IsUUID()
  gameId: string;

  @IsString()
  @Matches(/^[a-h][1-8][a-h][1-8][qrbn]?$/)  // UCI format
  move: string;

  @IsInt()
  @Min(0)
  timestamp: number;
}
```

Use `ValidationPipe` globally:
```typescript
app.useGlobalPipes(new ValidationPipe({
  whitelist: true,       // Strip unknown properties
  forbidNonWhitelisted: true,
  transform: true,       // Auto-transform types
}));
```

## Inter-Service Communication Matrix

| From → To | Method | When |
|-----------|--------|------|
| game-engine → clock-service | Redis Pub/Sub | Move made, need clock update |
| game-engine → anticheat | Bull Queue | Post-move analysis (async) |
| game-engine → rating-service | Bull Queue | Game ended, recalculate Elo |
| websocket-gateway → game-engine | Direct (same process or Redis) | Client sends move |
| matchmaking → game-engine | Redis Pub/Sub | Match found, create game |
| api-gateway → any service | HTTP (internal only) | REST queries from frontend |
| analytics → Redis Streams | Consumer Group | Read events asynchronously |

## Health Checks

Every service exposes:
```typescript
@Controller('health')
export class HealthController {
  @Get('live')
  liveness() { return { status: 'ok', timestamp: Date.now() }; }

  @Get('ready')
  async readiness() {
    // Check Redis, DB, dependencies
    return { status: 'ok', redis: true, db: true };
  }
}
```

## Error Handling

Use NestJS exception filters with structured errors:
```typescript
export class ServiceException extends HttpException {
  constructor(
    public readonly service: string,
    public readonly code: string,
    message: string,
    status: number = 500,
  ) {
    super({ service, code, message, timestamp: new Date().toISOString() }, status);
  }
}
```

## Module Registration Pattern

```typescript
@Module({
  imports: [
    RedisModule.forRootAsync({ useFactory: redisConfigFactory }),
    BullModule.registerQueue({ name: 'anticheat-analysis' }),
    PrismaModule,
    SharedTypesModule,
  ],
  controllers: [GameEngineController, HealthController],
  providers: [GameEngineService, MoveValidationService],
  exports: [GameEngineService],
})
export class GameEngineModule {}
```
