# Database Polyglot Strategy: CockroachDB + MongoDB + Redis

## Database Responsibilities

```
┌────────────────────────────────────────────────────────┐
│                    DATA STRATEGY                        │
├──────────────┬──────────────────┬──────────────────────┤
│    Redis     │   CockroachDB    │      MongoDB          │
│  (Hot Data)  │  (Relational)    │   (Documents)         │
├──────────────┼──────────────────┼──────────────────────┤
│ Active games │ Users            │ PGN game history      │
│ Live clocks  │ Payments         │ Anticheat reports     │
│ Sessions     │ Ratings/Elo      │ Analytics logs        │
│ Matchmaking  │ Friendships      │ Tournament brackets   │
│ Pub/Sub      │ Achievements     │ Chat history          │
│ Cache        │ Subscriptions    │ Engine analysis dumps  │
└──────────────┴──────────────────┴──────────────────────┘
```

### Rule: NEVER cross boundaries

- A service that reads from CockroachDB does NOT write to MongoDB in the same transaction
- Redis is the ONLY source of truth for active game state
- CockroachDB handles ACID transactions (payments, ratings)
- MongoDB handles write-heavy append-only documents

## Redis Patterns

### Game State (Hash)
```redis
HSET game:{gameId} fen "..." pgn "..." white_id "..." black_id "..."
                    white_time 600000 black_time 600000 turn "w"
                    status "active" version 1 created_at "..."
EXPIRE game:{gameId} 86400
```

### Clock State (Hash)
```redis
HSET clock:{gameId} white_ms 598000 black_ms 600000
                     last_move_at 1709123456789 active_color "b"
                     increment_ms 2000 time_control "10+2"
EXPIRE clock:{gameId} 86400
```

### Matchmaking Queue (Sorted Set)
```redis
ZADD matchmaking:rapid:{ratingBand} {timestamp} {playerId}
# Score = timestamp for FIFO within rating band
# Rating bands: 0-800, 800-1200, 1200-1600, 1600-2000, 2000+
```

### Session Cache
```redis
SET session:{userId} {JWT payload JSON} EX 3600
# Quick lookup without re-verifying JWT every time
```

### Pub/Sub Channels
```
game:{gameId}:update     → Move updates
game:{gameId}:clock      → Clock ticks
game:{gameId}:end        → Game termination
matchmaking:found        → Match paired
anticheat:alert          → Suspicious behavior detected
```

### Redis Streams (Analytics)
```redis
XADD analytics:moves * gameId g1 move e2e4 playerId p1 timestamp 1709123456
XADD analytics:games * gameId g1 event "game_end" result "1-0" moves 42
```

## CockroachDB / Prisma Schema

```prisma
model User {
  id        String   @id @default(uuid())
  username  String   @unique
  email     String   @unique
  password  String   // bcrypt hashed
  rating    Int      @default(1200)
  ratingDeviation Float @default(350)
  gamesPlayed Int    @default(0)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  whiteGames GameRecord[] @relation("WhitePlayer")
  blackGames GameRecord[] @relation("BlackPlayer")
  payments   Payment[]
}

model GameRecord {
  id          String   @id @default(uuid())
  whiteId     String
  blackId     String
  result      String   // "1-0", "0-1", "1/2-1/2"
  timeControl String   // "10+2", "5+0", etc.
  openingEco  String?  // ECO code
  moves       Int      // Total half-moves
  endReason   String   // "checkmate", "resign", "timeout", "draw"
  ratingWhite Int      // Rating at time of game
  ratingBlack Int
  ratingDelta Int      // How much rating changed
  playedAt    DateTime @default(now())

  white User @relation("WhitePlayer", fields: [whiteId], references: [id])
  black User @relation("BlackPlayer", fields: [blackId], references: [id])

  @@index([whiteId, playedAt])
  @@index([blackId, playedAt])
}

model Payment {
  id        String   @id @default(uuid())
  userId    String
  amount    Int      // cents
  currency  String   @default("USD")
  status    String   // "pending", "completed", "failed", "refunded"
  provider  String   // "stripe", "paypal"
  externalId String?
  createdAt DateTime @default(now())

  user User @relation(fields: [userId], references: [id])
}
```

### Prisma Best Practices
- Run `prisma generate` after ANY schema change
- Migrations: `prisma migrate dev` (local), `prisma migrate deploy` (prod)
- NEVER use raw SQL when Prisma can handle it
- Use transactions for multi-model updates (ratings + game record)

## MongoDB Collections

### Game History (PGN Archive)
```typescript
interface GamePGN {
  _id: ObjectId;
  gameId: string;          // Same as Redis game:{gameId}
  pgn: string;             // Full PGN notation
  fen_final: string;       // Final position
  moves: Array<{
    number: number;
    white: string;         // SAN notation
    black?: string;
    whiteClock: number;
    blackClock: number;
    whiteEval?: number;    // Stockfish eval (post-game)
    blackEval?: number;
  }>;
  metadata: {
    white: string;
    black: string;
    result: string;
    timeControl: string;
    opening: string;
    eco: string;
    date: Date;
  };
}
```

### Anticheat Reports
```typescript
interface AnticheatLog {
  _id: ObjectId;
  gameId: string;
  playerId: string;
  analyzedAt: Date;
  engine: 'stockfish' | 'lc0';
  depth: number;
  moves: Array<{
    moveNumber: number;
    playerMove: string;
    bestMove: string;
    centipawnLoss: number;
    moveTimeMs: number;
  }>;
  averageCPL: number;
  bestMoveRate: number;
  verdict: string;
  confidence: number;
}
```

## Data Flow: Game Lifecycle

```
1. MATCHMAKING
   Redis ZADD → pair players → Redis PUB matchmaking:found

2. GAME START
   Redis HSET game:{id} → Initial state
   Redis HSET clock:{id} → Initial clocks
   CockroachDB → Create GameRecord (status: "active")

3. DURING GAME
   Redis only → All moves, clocks, state changes
   Bull Queue → Async anticheat analysis

4. GAME END
   Redis → Final state read
   CockroachDB → Update GameRecord (result, ratings)
   MongoDB → Store full PGN + analysis
   Redis → DELETE game:{id}, clock:{id}

5. POST-GAME
   MongoDB → Deep analysis results
   CockroachDB → Rating history, stats
```
