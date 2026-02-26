# Testing Strategy for KOG Monorepo

## Testing Philosophy

> "If the game-engine has no tests, you don't have a game engine. You have a prayer."

Testing is NOT optional in this monorepo. The rule is simple:
- **New file → New `.spec.ts`** (no exceptions)
- **Bug fix → Test that reproduces the bug FIRST, then fix**
- **Refactor → Tests pass before AND after**

## Test Pyramid

```
         ╱╲
        ╱ E2E ╲          5% — Critical user flows only
       ╱────────╲
      ╱Integration╲      25% — Service interactions, Redis, queues
     ╱──────────────╲
    ╱   Unit Tests    ╲   70% — Pure logic, validators, services
   ╱────────────────────╲
```

## Unit Testing Patterns

### Testing a NestJS Service

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { GameEngineService } from './game-engine.service';
import { MoveValidationService } from './move-validation.service';
import { RedisService } from '../providers/redis.service';

describe('GameEngineService', () => {
  let service: GameEngineService;
  let redisService: jest.Mocked<RedisService>;
  let moveValidator: jest.Mocked<MoveValidationService>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        GameEngineService,
        {
          provide: RedisService,
          useValue: {
            hgetall: jest.fn(),
            multi: jest.fn().mockReturnValue({
              hset: jest.fn().mockReturnThis(),
              hincrby: jest.fn().mockReturnThis(),
              publish: jest.fn().mockReturnThis(),
              exec: jest.fn().mockResolvedValue([]),
            }),
          },
        },
        {
          provide: MoveValidationService,
          useValue: {
            validateMove: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get(GameEngineService);
    redisService = module.get(RedisService);
    moveValidator = module.get(MoveValidationService);
  });

  describe('processMove', () => {
    it('should reject illegal moves', async () => {
      moveValidator.validateMove.mockReturnValue({
        valid: false,
        reason: 'illegal_move',
      });

      await expect(
        service.processMove('game-1', 'player-1', 'e2e5'),
      ).rejects.toThrow('Illegal move');
    });

    it('should update Redis atomically on valid move', async () => {
      moveValidator.validateMove.mockReturnValue({
        valid: true,
        newFen: 'new-fen',
        san: 'e4',
        isCheckmate: false,
      });

      redisService.hgetall.mockResolvedValue({
        fen: 'starting-fen',
        turn: 'w',
        white_id: 'player-1',
      });

      await service.processMove('game-1', 'player-1', 'e2e4');

      expect(redisService.multi).toHaveBeenCalled();
    });

    it('should reject moves from wrong player', async () => {
      redisService.hgetall.mockResolvedValue({
        fen: 'some-fen',
        turn: 'w',
        white_id: 'player-1',
        black_id: 'player-2',
      });

      await expect(
        service.processMove('game-1', 'player-2', 'e7e5'),
      ).rejects.toThrow('Not your turn');
    });
  });
});
```

### Testing a WebSocket Gateway

```typescript
import { Test } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import { io, Socket } from 'socket.io-client';
import { GameGateway } from './game.gateway';

describe('GameGateway', () => {
  let app: INestApplication;
  let clientSocket: Socket;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      providers: [GameGateway, /* mocked deps */],
    }).compile();

    app = module.createNestApplication();
    await app.listen(0); // Random port
    const port = app.getHttpServer().address().port;

    clientSocket = io(`http://localhost:${port}`, {
      auth: { token: 'valid-jwt-token' },
    });

    await new Promise(resolve => clientSocket.on('connect', resolve));
  });

  afterAll(async () => {
    clientSocket.disconnect();
    await app.close();
  });

  it('should reject moves without auth', (done) => {
    const unauthSocket = io(serverUrl, { auth: {} });
    unauthSocket.on('connect_error', (err) => {
      expect(err.message).toContain('Unauthorized');
      done();
    });
  });

  it('should broadcast move to game room', (done) => {
    clientSocket.emit('game:move', { gameId: 'g1', move: 'e2e4' });
    clientSocket.on('game:update', (data) => {
      expect(data.m).toBe('e2e4');
      done();
    });
  });
});
```

### Testing Bull Queue Workers

```typescript
describe('AnticheatProcessor', () => {
  let processor: AnticheatProcessor;

  beforeEach(async () => {
    // Mock Stockfish process
    processor = new AnticheatProcessor(mockStockfish, mockRedis, mockLogger);
  });

  it('should analyze and return centipawn loss', async () => {
    mockStockfish.analyze.mockResolvedValue({
      bestMove: 'e2e4',
      score: { type: 'cp', value: 30 },
      depth: 20,
    });

    const job = createMockJob({
      gameId: 'g1',
      fen: 'starting-position',
      move: 'e2e4',
    });

    const result = await processor.analyze(job);
    expect(result.centipawnLoss).toBe(0); // Played best move
  });

  it('should respect circuit breaker when open', async () => {
    processor.circuitBreaker.forceOpen();

    const job = createMockJob({ gameId: 'g1' });
    const result = await processor.analyze(job);

    expect(result.skipped).toBe(true);
    expect(mockStockfish.analyze).not.toHaveBeenCalled();
  });
});
```

## Coverage Targets

| Service | Current | Target | Priority |
|---------|---------|--------|----------|
| game-engine | ~16% | 80% | 🔴 Phase 1 |
| clock-service | 0% | 80% | 🔴 Phase 1 |
| websocket-gateway | ~10% | 60% | 🟡 Phase 3 |
| anticheat-service | ~20% | 70% | 🟡 Phase 2 |
| matchmaking | ~30% | 70% | 🟡 Phase 2 |
| rating-service | ~15% | 70% | 🟡 Phase 2 |
| api-gateway | ~5% | 50% | 🟢 Phase 4 |

## CI Pipeline Integration

```yaml
# .github/workflows/test.yml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      redis:
        image: redis:7
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test -- --coverage --ci
      - run: npm run test:e2e
      - uses: codecov/codecov-action@v3
```

## Testing Anti-Patterns to AVOID

1. **Testing implementation, not behavior** — Don't test that a specific Redis command was called; test that the game state changed correctly.
2. **Flaky timing tests** — Use fake timers for clock-service tests, never `setTimeout` assertions.
3. **Shared state between tests** — Each test creates its own game state. `beforeEach` resets everything.
4. **No test for error paths** — Every `try/catch` in production code needs a test that hits the `catch`.
5. **Mocking everything** — Integration tests with real Redis (in Docker) catch bugs mocks miss.
