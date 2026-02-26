# Security & Anti-Cheat Pipeline

## Authentication Architecture

### JWT Flow

```
[Client] → POST /auth/login → [api-gateway] → [auth-service]
                                                     │
                                                     ▼
                                              Validate credentials
                                              Generate JWT (access + refresh)
                                                     │
                                                     ▼
[Client] ← { accessToken, refreshToken } ←──────────┘

[Client] → WebSocket connect (auth: { token: accessToken })
     │
     ▼
[websocket-gateway] → Verify JWT → Extract userId → Allow connection
```

### JWT Guard Implementation

```typescript
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private readonly jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = this.extractToken(request);

    if (!token) {
      throw new UnauthorizedException('Missing auth token');
    }

    try {
      const payload = this.jwtService.verify(token);
      request.user = payload;
      return true;
    } catch (error) {
      throw new UnauthorizedException('Invalid or expired token');
    }
  }

  private extractToken(request: Request): string | null {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : null;
  }
}
```

### WebSocket Auth Guard

```typescript
@Injectable()
export class WsJwtGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const client = context.switchToWs().getClient<Socket>();
    const token = client.handshake.auth?.token;

    if (!token) {
      client.disconnect();
      throw new WsException('Unauthorized');
    }

    try {
      const payload = this.jwtService.verify(token);
      client.data.userId = payload.sub;
      client.data.username = payload.username;
      return true;
    } catch {
      client.disconnect();
      throw new WsException('Invalid token');
    }
  }
}
```

### Critical: Guards MUST NOT be bypassed

Currently in the codebase, JWT guards are commented out in `game-engine` for development. This is a **CRITICAL security vulnerability** that must be fixed in Phase 1.

```typescript
// CURRENT (BROKEN — security hole)
// @UseGuards(JwtAuthGuard)
@Controller('game')
export class GameController { ... }

// CORRECT
@UseGuards(JwtAuthGuard)
@Controller('game')
export class GameController { ... }
```

## Anti-Cheat Pipeline

### Detection Layers

```
Layer 1: Real-time (in game-engine)
├── Move timing analysis (too fast = bot)
├── Server-side move validation (chess.js)
└── Session anomaly detection (multiple tabs, IP changes)

Layer 2: Post-move async (Bull Queue → Stockfish)
├── Centipawn loss analysis per move
├── Best-move match rate (>90% over 20+ moves = suspicious)
└── Move time vs position complexity correlation

Layer 3: Post-game deep (Bull Queue → Stockfish + Lc0)
├── Full game analysis with both engines
├── Statistical comparison against player's rating band
├── Pattern matching against known bot signatures
└── Cross-game behavioral profiling
```

### Centipawn Loss (CPL) Thresholds

| Rating Band | Expected Avg CPL | Flagging Threshold | Confidence Moves |
|------------|-------------------|---------------------|-----------------|
| < 1000 | 80-150 | < 20 avg over 20 moves | 25 |
| 1000-1500 | 40-80 | < 15 avg over 20 moves | 25 |
| 1500-2000 | 20-40 | < 10 avg over 25 moves | 30 |
| 2000-2500 | 10-25 | < 5 avg over 25 moves | 30 |
| 2500+ | 5-15 | Manual review only | N/A |

### Anti-Cheat Data Model

```typescript
interface AnticheatReport {
  gameId: string;
  playerId: string;
  playerRating: number;

  // Per-move analysis
  moves: Array<{
    moveNumber: number;
    playerMove: string;
    bestMove: string;
    centipawnLoss: number;
    moveTimeMs: number;
    positionComplexity: number; // Number of legal moves available
  }>;

  // Aggregate metrics
  averageCPL: number;
  bestMoveRate: number;        // % of moves matching engine's #1 choice
  top3Rate: number;            // % of moves matching engine's top 3
  suspiciousTimingCount: number; // Moves played in <1s in complex positions

  // Verdict
  verdict: 'clean' | 'suspicious' | 'flagged' | 'banned';
  confidence: number;          // 0-100
  reviewRequired: boolean;
}
```

### Rate Limiting

```typescript
// api-gateway rate limiting
@UseGuards(ThrottlerGuard)
@Throttle({ default: { limit: 60, ttl: 60000 } })  // 60 req/min general

// Stricter for auth endpoints
@Throttle({ default: { limit: 5, ttl: 60000 } })    // 5 req/min for login
@Post('auth/login')
async login() { ... }

// WebSocket event rate limiting (custom)
@SubscribeMessage('game:move')
async handleMove(client: Socket, data: MakeMoveDto) {
  const key = `ratelimit:ws:${client.data.userId}`;
  const count = await this.redis.incr(key);
  if (count === 1) await this.redis.expire(key, 1);
  if (count > 5) { // Max 5 moves per second (impossible in real chess)
    client.emit('error', { code: 'RATE_LIMITED' });
    return;
  }
  // Process move...
}
```

## Deprecated simple-auth Service

The `simple-auth` service is DEPRECATED and MUST be eliminated in Phase 1. It uses:
- Plain text password comparison (no bcrypt)
- Session tokens stored in memory (lost on restart)
- No rate limiting on login attempts
- No token expiration

Replace with proper JWT auth through `api-gateway` + `auth-service`.

## Input Sanitization

Every WebSocket event and HTTP endpoint validates input with `class-validator`:

```typescript
// NEVER trust client data
export class MakeMoveDto {
  @IsUUID('4')
  gameId: string;

  @IsString()
  @Length(4, 5)
  @Matches(/^[a-h][1-8][a-h][1-8][qrbn]?$/)
  move: string;

  @IsInt()
  @Min(0)
  @Max(Date.now() + 60000) // Can't be more than 1 min in the future
  timestamp: number;
}
```
