# Real-Time Architecture: WebSocket, Socket.IO & Redis Adapter

## Architecture Overview

```
[Client Browser]
       │
       │ Socket.IO (WSS)
       ▼
[websocket-gateway]  ──Redis Adapter──  [websocket-gateway instance 2]
       │                                         │
       │ Redis Pub/Sub                          │
       ▼                                         ▼
[game-engine]  ←──Redis──→  [clock-service]
       │
       │ Bull Queue (async)
       ▼
[anticheat-service]
```

## Socket.IO with Redis Adapter

For horizontal scaling, ALL Socket.IO instances share state via Redis:

```typescript
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';

const pubClient = createClient({ url: REDIS_URL });
const subClient = pubClient.duplicate();

server.adapter(createAdapter(pubClient, subClient));
```

This ensures:
- Room membership is shared across instances
- Events broadcast to ALL connected clients regardless of which instance they're on
- Sticky sessions are NOT required (but recommended for performance)

## WebSocket Event Flow (Game Move)

```
1. Client sends:    socket.emit('game:move', { gameId, move, timestamp })
2. Gateway receives: @SubscribeMessage('game:move')
3. Gateway validates: JWT token + DTO validation
4. Gateway calls:    gameEngine.processMove(gameId, playerId, move)
5. Engine validates:  chess.js validates legality
6. Engine stores:     Redis HSET game:{id} state (atomic)
7. Engine publishes:  Redis PUB game:{id}:update {delta}
8. Clock updates:     clock-service receives pub, recalculates time
9. Gateway broadcasts: socket.to(gameRoom).emit('game:update', delta)
10. Anticheat queued:  Bull queue receives move for async analysis
```

## Payload Optimization

### BAD: Full state on every move
```json
{
  "fen": "rnbqkbnr/pppppppp/8/8/4P3/8/PPPP1PPP/RNBQKBNR b KQkq e3 0 1",
  "pgn": "1. e4",
  "moves": ["e2e4"],
  "whiteTime": 598000,
  "blackTime": 600000,
  "capturedPieces": [],
  "moveHistory": [{"from": "e2", "to": "e4", "piece": "p"}]
}
```
~350 bytes per move, grows linearly.

### GOOD: Delta only
```json
{
  "m": "e2e4",
  "wt": 598000,
  "bt": 600000,
  "mn": 1
}
```
~60 bytes, constant size.

Full state is ONLY sent on:
- Initial connection
- Reconnection
- Explicit state request

## Reconnection Flow

```
1. Client detects disconnect (Socket.IO auto-detect)
2. Socket.IO attempts reconnection with exponential backoff
3. On reconnect:
   a. Client sends: socket.emit('game:reconnect', { gameId, lastMoveNumber })
   b. Gateway checks: Is game still active in Redis?
   c. If yes: Fetch full state from Redis HGETALL game:{id}
   d. Send delta: Only moves since lastMoveNumber
   e. Client rebuilds board state from delta
   f. Resume normal flow
4. If game ended during disconnect:
   a. Send game result
   b. Redirect to post-game screen
```

### Critical: Reconnection MUST NOT corrupt game state

```typescript
// Atomic reconnection check
const gameState = await this.redis.multi()
  .hgetall(`game:${gameId}`)
  .get(`game:${gameId}:version`)
  .exec();

// If client version matches, send delta
// If client version is behind, send full state from their version forward
// If game no longer exists, send termination event
```

## Room Management

```typescript
// Player joins game room
socket.join(`game:${gameId}`);

// Spectators join spectator room
socket.join(`spectate:${gameId}`);

// Player-specific events (clock warnings)
socket.join(`player:${playerId}`);

// Cleanup on disconnect
socket.on('disconnect', () => {
  this.handleDisconnect(socket.data.playerId, socket.data.gameId);
});
```

## Lag Compensation in Clock Service

```typescript
interface ClockUpdate {
  gameId: string;
  playerId: string;
  moveTimestamp: number;      // Client-reported timestamp
  serverReceived: number;     // Server-received timestamp
  lagCompensation: number;    // Estimated network delay
  adjustedTime: number;       // Final time after compensation
}

// Lag estimation: average of last 5 round-trip measurements
// Compensation: max(0, min(estimatedLag, MAX_COMPENSATION_MS))
// MAX_COMPENSATION_MS = 500 (never compensate more than 500ms)
```

## Scaling Considerations

| Metric | Target | Strategy |
|--------|--------|----------|
| Concurrent connections | 10,000+ | Redis Adapter + horizontal pods |
| Move latency (server) | < 50ms | Redis pipeline, no DB reads in hot path |
| Broadcast latency | < 100ms | Delta-only payloads, room-scoped |
| Reconnection time | < 2s | Full state in Redis, no DB fetch |
| Memory per game | < 5KB | Compressed FEN + clock + metadata |
