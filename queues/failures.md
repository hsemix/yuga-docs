# Failures and Replay

Yuga exposes two failure-management paths. Choose commands according to where the failed delivery is retained.

| Storage | Inspect | Recover | Identifier |
| --- | --- | --- | --- |
| Database transport's failed message rows | `queue:failed` | `queue:retry` | Database row ID from `failure_id` |
| Separate failure store, used by RoadRunner | `queue:failures` | `queue:replay` | 64-character failure-store ID |

Neither failure ID is interchangeable with the portable envelope `job_id` or RoadRunner task ID.

## Database transport failures

```bash
php yuga queue:failed --connection=database --queue=reports --limit=50 --offset=0
php yuga queue:retry 12 --connection=database --delay=5
php yuga queue:forget 12 --connection=database
```

Use the actual `failure_id` printed by `queue:failed`. Retry atomically moves only a currently failed row back to waiting, preserves its envelope identity/body, resets attempts to zero, and clears the failure reason. Forget deletes only that failed row. These commands require `FailedJobTransportInterface`.

## RoadRunner terminal failures

At the attempt limit, the consumer records the envelope, delivery ID, pipeline, attempt count, and reason before sending a terminal nack. If storage fails, the consumer does not intentionally discard the task by nacking it. Actual crash/redelivery behaviour remains dependent on the backend.

Failure records deduplicate by source plus delivery ID. Use a stable source identifying the application/deployment/connection, such as `billing-production-roadrunner`. Distinct deployments should not accidentally share a source. The committed test configuration uses `fntyuga-roadrunner-test`; it is an example identity.

```bash
php yuga queue:failures --source=fntyuga-roadrunner-test
php yuga queue:failures --source=fntyuga-roadrunner-test --limit=20 --offset=20
php yuga queue:failures FAILURE_ID
```

Output includes failure ID, source, delivery ID, destination, job ID, type, encoding, attempts, reason, and UTC failure time. Raw bodies are not printed or deserialized. `--source` filters listings; lookup by ID calls `find()` directly and is not source-filtered.

## Register a replay destination

`queue.replayers` maps each source to a `FailureReplayer`. Adapt its binding in your application/provider to your own source names and publishers:

```php
use Yuga\Queue\Failure\FailureReplayer;

$app->singleton('queue.replayers', function () {
    return [
        'billing-production-roadrunner' => new FailureReplayer(
            app('queue.failures'),
            app('queue.manager')->connection('roadrunner'),
            'billing-production-roadrunner'
        ),
    ];
});
```

The committed provider currently hardcodes a replayer for `fntyuga-roadrunner-test`. Merely changing the connection's `failure_source` does not automatically create a matching replayer.

The stored RoadRunner destination is the actual pipeline name. Include a mapping from that name to the pipeline in the publisher's `pipelines` settings, for example `'yuga-test' => 'yuga-test'`, so replay can publish it.

## Replay a retained failure

```bash
php yuga queue:replay FAILURE_ID --source=billing-production-roadrunner --delay=0
```

Replay atomically claims a pending record, publishes its original envelope, then marks it replayed. The original failure remains for inspection. RoadRunner creates a new task ID and starts `yuga-attempt` at 1, while the Yuga job ID stays the same. A replayed job that fails again creates a new failure record for the new delivery.

Successful replay means publication, not successful execution. A repeated request for the same failure is refused with:

```text
Failure not found for this source, already claimed, or already replayed.
```

If publication or marking completion fails after a claim, the record stays `replaying`. It is deliberately not automatically released or expired: the backend might already have accepted the task. Inspect backend delivery and the record before any manual recovery to avoid duplicate publication. This is not an exactly-once guarantee.

## Current limits

Malformed envelope or attempt metadata escapes the consumer before normal execution handling. A raw-message quarantine facility is not implemented yet. There is also no separate failure-store forget/prune CLI in this API; `queue:forget` applies to transport-managed failures only.
