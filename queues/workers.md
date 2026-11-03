# Workers and Lifecycle

## Polling worker

```bash
php yuga queue:work database --queue=reports
```

The connection argument is optional and falls back to `queue_transport.default`. It must support `QueueTransportInterface`; the RoadRunner publisher does not. Use [RoadRunner's dedicated consumer](roadrunner.md) for that connection.

| Option | Default | Behaviour |
| --- | --- | --- |
| `--queue` | `default` | One queue to poll |
| `--once` | off | Make one reservation attempt, then exit |
| `--stop` | off | Exit when no job is currently available |
| `--sleep` | `3` | Idle wait in seconds |
| `--max-jobs` | `0` | Reservations processed before exit; zero disables |
| `--max-time` | `0` | Runtime limit in seconds, checked between jobs; zero disables |
| `--memory` | `128` | Allocated-memory threshold in MB, checked between jobs; zero disables |

```bash
php yuga queue:work database --queue=reports --once
php yuga queue:work database --queue=reports --stop
php yuga queue:work database --max-jobs=100 --max-time=1800 --memory=256
```

These limits do not interrupt a running job. `--max-jobs` counts processed reservations, including released or failed attempts. `--stop` may exit while future delayed jobs still exist. `--once` can exit without processing anything when the queue is currently empty.

## Reservation and retry behaviour

A database reservation atomically claims an eligible row, increments attempts, and creates a lease token. Successful execution acknowledges the reservation. An exception releases it with the configured delay until `max_attempts` is reached; terminal failure retains the envelope and reason in the transport.

An expired reservation becomes available for recovery. An old owner cannot acknowledge, release, fail, or renew it after ownership expires or changes. A lost reservation emits `queue:reservation-lost` and causes the worker to exit with failure. Settlement/backend errors escape rather than being treated as another execution failure.

Set the reservation duration longer than expected execution time or renew it cooperatively through `JobContext`. Expiration can permit another worker to execute the same job even if the original process is still running. There is no automatic heartbeat or hard execution timeout.

## Lifecycle listeners

Register a listener after the queue services are loaded:

```php
app('queue.reporter')->listen(
    static function (string $event, array $context): void {
        error_log(json_encode([
            'event' => $event,
            'job_id' => $context['job_id'],
            'type' => $context['type'],
            'attempt' => $context['attempt'],
        ], JSON_THROW_ON_ERROR));
    }
);
```

| Event | Meaning |
| --- | --- |
| `queue:processing` | An execution attempt is about to decode and run |
| `queue:successful` | Execution and acknowledgement succeeded |
| `queue:retrying` | The failed attempt was released or requeued |
| `queue:failure` | Terminal failure was settled |
| `queue:callback-failure` | The job's terminal callback threw |
| `queue:reservation-lost` | Polling-worker ownership expired or was lost |

Common context includes `job_id`, `type`, `encoding`, and `attempt`. Failure-related context carries the `Throwable` under `exception`. RoadRunner also includes task, pipeline, and source metadata; terminal notifications include `failure_id`.

Listener exceptions are caught and logged individually. They do not change the job's outcome or stop subsequent listeners. Lifecycle reporting and terminal callbacks are best effort, not a durable event bus.

## Graceful restart

```bash
php yuga queue:restart
```

The command changes a shared restart token at `storage/framework/queue.restart`. Existing `queue:work` processes compare it between reservations and exit after current work. A supervisor must start replacement workers; the command itself does not launch them. Workers must see the same writable storage path. This file mechanism is scoped to that shared filesystem, not automatically to every host in a deployment.

**The current RoadRunner consumer does not read this token.** `queue:restart` does not yet recycle RoadRunner Jobs workers or selectively reset their pool. Do not rely on it to reload RoadRunner queue code.

On platforms with PCNTL, the polling command handles SIGTERM/SIGINT cooperatively. Without PCNTL, including Windows PHP environments, the polling loop and restart-token mechanism remain available; Unix signal handling is not promised there. External runtime/binary support is separate.

## Monitoring

```bash
php yuga queue:monitor --connection=database --queue=reports
```

The command requires `MonitorableTransportInterface`. Its snapshot contains ready, delayed, reserved, expired, failed, completed, and total counts; it is not a continuous dashboard. RoadRunnerPublisher does not implement this capability.

The current database class has a `statistics()` method but does not declare `MonitorableTransportInterface` in its `implements` list. With that exact revision, the command reports unsupported monitoring even for the database connection. The method alone does not satisfy the command's capability check; this remains an implementation gap rather than a configuration option.
