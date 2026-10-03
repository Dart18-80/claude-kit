---
name: bullmq-patterns
description: Background jobs with BullMQ on Redis in NestJS backends — queue registration, processors (WorkerHost), retries with backoff, idempotent job IDs, concurrency and rate limits, flows, graceful shutdown, and testing. Use when adding or reviewing a queue, worker, scheduled/repeatable job, or anything using @nestjs/bullmq, bullmq, or legacy @nestjs/bull.
metadata:
  origin: claude-kit (original)
---

# BullMQ Patterns (NestJS + Redis)

Patterns for reliable background processing with BullMQ in a NestJS service.

## When to Activate

- Adding a queue, a worker/processor, or a repeatable/scheduled job
- Moving slow work (emails, webhooks, AI calls, file processing, ETL) out of the request path
- Reviewing retry, idempotency, or shutdown behavior of existing workers
- Migrating from legacy `@nestjs/bull` (Bull v3) to `@nestjs/bullmq`

## Library Choice

- **New code: `bullmq` + `@nestjs/bullmq`.** Bull v3 (`bull` + `@nestjs/bull`) is in maintenance mode.
- Legacy Bull uses `@Process()` methods on a `@Processor()` class; BullMQ uses a class that extends `WorkerHost` and implements `process()`. Do not mix the two packages in one module.
- Before writing code, read the installed versions in `package.json` and follow the API of that version.

## Module Setup

Register the Redis connection once, then register each queue in the module that owns it.

```typescript
// app.module.ts
BullModule.forRootAsync({
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    connection: {
      host: config.getOrThrow('REDIS_HOST'),
      port: config.getOrThrow<number>('REDIS_PORT'),
      password: config.get('REDIS_PASSWORD'),
    },
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: 'exponential', delay: 1_000 },
      removeOnComplete: { count: 1_000 },
      removeOnFail: { age: 7 * 24 * 3600 },
    },
  }),
}),

// notifications.module.ts
BullModule.registerQueue({ name: NOTIFICATIONS_QUEUE }),
```

Rules:

- Queue names are constants exported from one file, never string literals scattered across modules.
- Always set `removeOnComplete` / `removeOnFail`. Without them Redis memory grows forever.
- Validate Redis env vars at boot with the same config schema as the rest of the app.

## Producing Jobs

```typescript
@Injectable()
export class NotificationsService {
  constructor(@InjectQueue(NOTIFICATIONS_QUEUE) private readonly queue: Queue<SendEmailJob>) {}

  async sendWelcome(userId: string) {
    await this.queue.add('send-welcome', { userId }, { jobId: `welcome:${userId}` });
  }
}
```

- **Payloads are small and serializable**: IDs and parameters, not entities, buffers, or class instances. The worker re-reads current state from the database.
- **Type the payload** (`Queue<SendEmailJob>`, `Job<SendEmailJob>`) and keep the type next to the queue constant.
- **Use a deterministic `jobId` for idempotency** when the same event can be emitted twice. BullMQ ignores an `add` whose `jobId` already exists.
- Enqueue **after** the database transaction commits. Enqueuing inside the transaction lets a worker read data that later rolls back. For strict guarantees use an outbox table that a poller turns into jobs.

## Processing Jobs

```typescript
@Processor(NOTIFICATIONS_QUEUE, { concurrency: 5 })
export class NotificationsProcessor extends WorkerHost {
  private readonly logger = new Logger(NotificationsProcessor.name);

  constructor(private readonly mailer: MailerService, private readonly users: UsersRepository) {
    super();
  }

  async process(job: Job<SendEmailJob>): Promise<void> {
    switch (job.name) {
      case 'send-welcome':
        return this.sendWelcome(job);
      default:
        throw new UnrecoverableError(`Unknown job name: ${job.name}`);
    }
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job, error: Error) {
    this.logger.error({ jobId: job.id, name: job.name, attempt: job.attemptsMade, err: error.message });
  }
}
```

- **Every processor must be idempotent.** Retries and stalled-job recovery mean a job can run more than once. Check-then-act against the DB (e.g. `welcomeSentAt IS NULL`) or use unique constraints.
- **Throw to retry; throw `UnrecoverableError` to fail without retry** (validation errors, missing records, unknown job names).
- Do not swallow errors with `try/catch` that only logs — the job is then marked completed.
- Long jobs: call `job.updateProgress()` and keep the work under the lock duration, or split it into smaller jobs.
- Keep processors thin: they call application services; business logic stays testable without Redis.

## Concurrency, Rate Limits, Scheduling

- `concurrency` is per worker process. Total parallelism = concurrency × number of instances.
- Respect third-party limits with the worker `limiter` option, e.g. `{ max: 10, duration: 1_000 }`.
- Repeatable / scheduled jobs: use the scheduler API of the installed BullMQ version (`upsertJobScheduler` in recent versions, `repeat` options in older ones) with a **stable scheduler ID** so deploys don't create duplicates.
- Parent/child pipelines: use `FlowProducer` instead of enqueuing the next step from inside a processor.

## Running Workers

- In production, consider running workers as a separate process or deployment (same codebase, different entrypoint) so API latency doesn't compete with heavy jobs.
- Call `app.enableShutdownHooks()` so workers close gracefully and in-flight jobs aren't left stalled on deploy.
- Use a Redis instance with `maxmemory-policy noeviction` for queues. Evicting keys corrupts queue state.
- Expose queue health (waiting / active / failed counts) to monitoring; alert on a growing failed or waiting count.

## Testing

- **Unit-test processors** by instantiating the class with mocked dependencies and calling `process()` with a fake `Job` object.
- **Unit-test producers** by overriding the queue provider:

```typescript
const module = await Test.createTestingModule({
  providers: [
    NotificationsService,
    { provide: getQueueToken(NOTIFICATIONS_QUEUE), useValue: { add: jest.fn() } },
  ],
}).compile();
```

- **Integration tests** run against a real Redis (Docker / Testcontainers) and use a unique queue prefix per test run; drain and close queues and workers in `afterAll`.

## Review Checklist

- [ ] Queue name is a shared constant; payload type is declared
- [ ] Payload holds IDs/params only
- [ ] `attempts`, `backoff`, `removeOnComplete`, `removeOnFail` are set
- [ ] Processor is idempotent; non-retryable errors use `UnrecoverableError`
- [ ] Jobs are enqueued after the DB transaction commits
- [ ] Repeatable jobs use a stable ID
- [ ] Shutdown hooks are enabled; failures are logged with job id and attempt
- [ ] Tests mock the queue via `getQueueToken`

## Related

- Skill: `redis-patterns` (this plugin) — caching, locks, rate limiting
- Skill: `nestjs-patterns` (stack-nestjs plugin) — module, provider, and config conventions
