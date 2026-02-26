# Bull Queue Resilience Patterns

## Queue Architecture in KOG

```
[game-engine]
     │
     ├──→ anticheat-analysis (Stockfish)    [HIGH priority]
     ├──→ anticheat-neural (Lc0)            [LOW priority]
     ├──→ rating-update (Elo recalc)        [MEDIUM priority]
     └──→ analytics-ingest (stats)          [LOW priority]

[matchmaking]
     └──→ match-pairing (ELO pool)          [HIGH priority]
```

## Queue Configuration Template

```typescript
BullModule.registerQueue({
  name: 'anticheat-analysis',
  redis: {
    host: process.env.REDIS_HOST,
    port: parseInt(process.env.REDIS_PORT),
    maxRetriesPerRequest: 3,
  },
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 2000,       // 2s, 4s, 8s
    },
    removeOnComplete: {
      age: 3600,         // Keep completed jobs for 1 hour
      count: 1000,       // Keep max 1000 completed
    },
    removeOnFail: false, // NEVER auto-remove failed — needs investigation
    timeout: 30000,      // 30s max per job
  },
});
```

## Worker Pattern with Circuit Breaker

```typescript
@Processor('anticheat-analysis')
export class AnticheatProcessor {
  private circuitBreaker = new CircuitBreaker({
    threshold: 5,        // 5 failures to open
    timeout: 60000,      // 60s before half-open
  });

  @Process('analyze-move')
  async analyze(job: Job<AnalyzeMovePayload>) {
    if (this.circuitBreaker.isOpen()) {
      // Don't waste resources, just log and skip
      this.logger.warn(`Circuit open, skipping analysis for game ${job.data.gameId}`);
      return { skipped: true, reason: 'circuit_open' };
    }

    try {
      const result = await this.stockfish.analyze(job.data.fen, {
        depth: job.data.priority === 'flagged' ? 25 : 20,
        timeout: 15000,
      });

      this.circuitBreaker.recordSuccess();
      return result;
    } catch (error) {
      this.circuitBreaker.recordFailure();
      throw error; // Bull will retry based on backoff config
    }
  }

  @OnQueueFailed()
  async onFailed(job: Job, error: Error) {
    this.logger.error(`Job ${job.id} failed: ${error.message}`, {
      gameId: job.data.gameId,
      attempt: job.attemptsMade,
      maxAttempts: job.opts.attempts,
    });

    if (job.attemptsMade >= job.opts.attempts) {
      // Move to Dead Letter Queue
      await this.dlqQueue.add('failed-analysis', {
        originalJob: job.data,
        error: error.message,
        failedAt: new Date().toISOString(),
      });
    }
  }
}
```

## Dead Letter Queue (DLQ)

Jobs that fail all retries go to DLQ for manual investigation:

```typescript
BullModule.registerQueue({
  name: 'dead-letter-queue',
  defaultJobOptions: {
    removeOnComplete: false,  // Keep forever until manually resolved
    removeOnFail: false,
  },
});
```

DLQ consumer runs a dashboard or alerts:
```typescript
@Processor('dead-letter-queue')
export class DLQProcessor {
  @Process('failed-analysis')
  async handleFailedAnalysis(job: Job) {
    // Log to incident tracking
    // Send alert (Slack, email, etc.)
    // Store for post-mortem
    await this.incidentService.create({
      type: 'anticheat-failure',
      data: job.data,
      severity: 'P2',
    });
  }
}
```

## Concurrency and Resource Limits

```typescript
@Processor('anticheat-analysis')
export class AnticheatProcessor implements OnModuleInit {
  // Limit concurrent Stockfish processes
  static readonly MAX_CONCURRENT = 4;

  onModuleInit() {
    // Process max 4 jobs simultaneously
    this.worker.concurrency = AnticheatProcessor.MAX_CONCURRENT;
  }
}
```

### Resource Budget per Queue

| Queue | Max Concurrent | Memory/Job | Timeout | Priority |
|-------|---------------|------------|---------|----------|
| anticheat-analysis | 4 | 256MB | 30s | HIGH |
| anticheat-neural | 2 | 512MB | 60s | LOW |
| rating-update | 8 | 64MB | 5s | MEDIUM |
| analytics-ingest | 4 | 128MB | 10s | LOW |
| match-pairing | 4 | 32MB | 5s | HIGH |

## Rate Limiting

Prevent queue flooding during peak hours:

```typescript
BullModule.registerQueue({
  name: 'anticheat-analysis',
  limiter: {
    max: 100,         // Max 100 jobs
    duration: 60000,  // Per 60 seconds
  },
});
```

## Monitoring

Essential Bull metrics to expose to Prometheus:

```typescript
// Queue health metrics
const metrics = {
  waiting: await queue.getWaitingCount(),
  active: await queue.getActiveCount(),
  completed: await queue.getCompletedCount(),
  failed: await queue.getFailedCount(),
  delayed: await queue.getDelayedCount(),
};
```

Alerts:
- Waiting > 100: Queue is backing up
- Failed > 10 in 5 min: Something is broken
- Active = MaxConcurrent for 5+ min: Workers are stuck
