# Chess Engine Integration: chess.js, Stockfish, Lc0

## chess.js — Move Validation (Synchronous, In-Process)

chess.js is the AUTHORITATIVE source of move legality. Every move passes through it BEFORE being accepted.

### Usage Pattern

```typescript
import { Chess } from 'chess.js';

class MoveValidationService {
  validateMove(fen: string, move: string): MoveResult {
    const chess = new Chess(fen);

    // Attempt the move
    const result = chess.move(move, { strict: true });

    if (!result) {
      return { valid: false, reason: 'illegal_move' };
    }

    return {
      valid: true,
      newFen: chess.fen(),
      isCheck: chess.isCheck(),
      isCheckmate: chess.isCheckmate(),
      isDraw: chess.isDraw(),
      isStalemate: chess.isStalemate(),
      isThreefoldRepetition: chess.isThreefoldRepetition(),
      isInsufficientMaterial: chess.isInsufficientMaterial(),
      capturedPiece: result.captured || null,
      san: result.san,
    };
  }
}
```

### Performance Considerations

- chess.js operations are SYNCHRONOUS and CPU-bound
- Creating a new Chess instance per validation is ~0.1ms (acceptable)
- Loading FEN and validating a single move: ~0.2ms average
- For analysis of ALL legal moves: ~2-5ms (avoid in hot path)
- NEVER run batch analysis in the game-engine event loop — delegate to Bull queue

### Event Loop Protection

```typescript
// BAD: Blocking analysis in game move handler
async processMove(gameId: string, move: string) {
  const analysis = this.analyzeAllLines(fen, 20); // Blocks 50ms+
  await this.broadcast(gameId, { move, analysis });
}

// GOOD: Validate only, queue analysis
async processMove(gameId: string, move: string) {
  const result = this.validateMove(fen, move);    // 0.2ms
  if (result.valid) {
    await this.broadcast(gameId, { move: result });
    await this.anticheatQueue.add('analyze', { gameId, fen, move }); // Async
  }
}
```

## Stockfish — Deep Analysis (Asynchronous, Worker)

Stockfish runs as a separate process/container. Communication via Bull Queue.

### Architecture

```
[game-engine] → Bull Queue → [anticheat-worker] → Stockfish process → Results → Redis
```

### Worker Implementation

```typescript
@Processor('anticheat-analysis')
class StockfishWorker {
  private engine: StockfishProcess;

  @Process('analyze-move')
  async analyzeMove(job: Job<AnalyzePayload>) {
    const { fen, move, gameId, moveNumber } = job.data;

    const analysis = await this.engine.analyze(fen, {
      depth: 20,           // Depth 20 for move analysis
      multipv: 3,          // Top 3 lines
      timeout: 10000,      // 10s max per position
    });

    const evaluation = {
      gameId,
      moveNumber,
      bestMove: analysis.bestMove,
      playerMove: move,
      centipawnLoss: this.calculateCPLoss(analysis, move),
      isBestMove: analysis.bestMove === move,
      evaluation: analysis.score,
      depth: analysis.depth,
    };

    await this.redis.hset(`analysis:${gameId}`, moveNumber.toString(), JSON.stringify(evaluation));

    return evaluation;
  }
}
```

### Resource Management

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Max concurrent workers | 4 per container | CPU-bound, avoid OOM |
| Analysis depth | 20 (standard), 25 (flagged) | Balance accuracy vs time |
| Timeout per position | 10s standard, 30s deep | Prevent worker stall |
| Memory per worker | 256MB max | Stockfish hash tables |
| Queue retry policy | 3 retries, exponential backoff | Resilience |
| Dead Letter Queue | After 3 failures | Don't lose data |

## Lc0 — Neural Network Analysis

Lc0 (Leela Chess Zero) provides a neural network-based evaluation complementary to Stockfish.

### Use Cases
- **Anti-cheat correlation:** If a player matches Stockfish 95%+ but not Lc0, suspicious pattern
- **Post-game analysis:** Offer "engine perspective" from both classical and neural engines
- **Cheater profiling:** Build centipawn-loss distributions across both engines

### Resource Considerations
- Lc0 is GPU-intensive (if using CUDA) or slower on CPU
- Run as a SEPARATE queue from Stockfish
- Priority: Stockfish queue > Lc0 queue (Lc0 is enrichment, not critical path)

## Game State in Redis

```
# Active game hash
HSET game:{gameId} fen "rnbqkbnr/pppppppp/8/8/4P3/8/PPPP1PPP/RNBQKBNR b KQkq e3 0 1"
HSET game:{gameId} pgn "1. e4"
HSET game:{gameId} white_id "uuid-white"
HSET game:{gameId} black_id "uuid-black"
HSET game:{gameId} white_time 598000
HSET game:{gameId} black_time 600000
HSET game:{gameId} turn "b"
HSET game:{gameId} move_count 1
HSET game:{gameId} status "active"
HSET game:{gameId} version 1

# TTL: 24 hours for active games, cleanup old ones
EXPIRE game:{gameId} 86400
```

### Atomic Move Processing

```typescript
// Use Redis MULTI/EXEC for atomic game state updates
const pipeline = this.redis.multi();
pipeline.hset(`game:${gameId}`, 'fen', newFen);
pipeline.hset(`game:${gameId}`, 'pgn', updatedPgn);
pipeline.hset(`game:${gameId}`, `${turn}_time`, newTime);
pipeline.hset(`game:${gameId}`, 'turn', nextTurn);
pipeline.hincrby(`game:${gameId}`, 'move_count', 1);
pipeline.hincrby(`game:${gameId}`, 'version', 1);
pipeline.publish(`game:${gameId}:update`, JSON.stringify(delta));
const results = await pipeline.exec();
```
