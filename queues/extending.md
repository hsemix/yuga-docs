# Custom Adapters and Codecs

Queue delivery and payload encoding are independent extension points. A Redis adapter can carry the same envelopes as the database transport; a new codec does not require changing every transport.

## Choose the adapter contract

All delivery adapters implement `Yuga\Queue\Contracts\QueuePublisherInterface`:

```php
public function publish(
    \Yuga\Queue\Payload\JobEnvelope $envelope,
    string $queue = 'default',
    int $delaySeconds = 0
): void;
```

Preserve envelope ID, type, version, encoding, and raw body exactly. Throw on backend failure or unsupported delay instead of silently dropping requirements. RoadRunner uses this contract with a dedicated consumer.

To participate in `queue:work`, also implement `QueueTransportInterface`, which extends publication with:

```php
public function reserve(
    string $queue = 'default',
    int $reservationSeconds = 60
): ?\Yuga\Queue\Transport\JobReservation;

public function acknowledge(\Yuga\Queue\Transport\JobReservation $reservation): bool;

public function release(
    \Yuga\Queue\Transport\JobReservation $reservation,
    int $delaySeconds = 0
): bool;

public function fail(
    \Yuga\Queue\Transport\JobReservation $reservation,
    string $reason
): bool;
```

Return `null` when nothing is available. Claim atomically before granting execution, increment the attempt count on successful reservation, and make expired unacknowledged work recoverable. A reservation contains its envelope, attempt number, and an opaque adapter receipt carrying ownership information.

Acknowledgement, release, failure, and renewal must check current ownership. Return `false` for expired/lost ownership and throw for backend errors. A stale consumer must not settle a newer reservation. Release preserves the envelope and attempt count; terminal failure removes normal delivery eligibility and retains the failure details.

For a Redis implementation, these guarantees typically require an atomic operation over ready, reserved, and delayed state, plus recovery of expired leases. A plain list pop without recovery and ownership checks does not implement the full polling contract.

## Register a driver

In a package or application provider after queue services are available:

```php
app('queue.manager')->extend(
    'redis',
    static function (array $settings) {
        return new \Vendor\Queue\RedisTransport(
            app('redis'),
            $settings['prefix'] ?? 'queues:'
        );
    }
);
```

This example refers to a developer-supplied adapter and Redis service, not built-in Yuga classes. Add a connection:

```php
'redis-main' => [
    'driver' => 'redis',
    'prefix' => 'myapp:queues:',
],
```

Factories receive the selected connection's settings and must return `QueuePublisherInterface`. Connections are created lazily and cached by connection name. Driver names cannot be registered twice. Register the driver before its first connection is resolved.

```php
$publisher = app('queue.manager')->connection('redis-main');
$transport = app('queue.manager')->transport('redis-main');
```

`connection()` accepts publication-only adapters. `transport()` additionally checks polling support and throws `UnsupportedCapabilityException` when absent.

## Optional capabilities

All interfaces below are in `Yuga\Queue\Contracts`:

| Interface | Behaviour exposed |
| --- | --- |
| `RenewableTransportInterface` | `renew(reservation, additionalSeconds)` extends an owned unexpired lease |
| `FailedJobTransportInterface` | `failedJobs`, `findFailed`, `retryFailed`, `forgetFailed` enable transport-managed failure commands |
| `MonitorableTransportInterface` | `statistics(?queue)` enables `queue:monitor` |
| `FailureStoreInterface` | `record(...)` persists a terminal failure, deduplicated by source and delivery ID |
| `InspectableFailureStoreInterface` | Adds `find` and paginated `listing` |
| `ReplayableFailureStoreInterface` | Adds atomic `claimReplay` and `markReplayed` |

Declare supported interfaces explicitly. A similarly named method alone will not pass capability checks. Failure stores are independent of delivery adapters and can retain failures from externally managed consumers.

## Add a codec

Implement `Yuga\Queue\Payload\PayloadCodecInterface`:

```php
public function name(): string;
public function encode(mixed $value): string;
public function decode(string $payload): mixed;
```

Register your implementation:

```php
app('queue.codecs')->register(new \Vendor\Queue\MessagePackCodec());
```

Use a unique stable encoding name and register it on producers and consumers before use. Do not silently substitute another codec when a name is unknown. Then publish with its actual name:

```php
app('queue.dispatcher')->publish(
    type: 'reports.generate.v1',
    payload: ['report_id' => 42],
    connection: 'database',
    queue: 'reports',
    encoding: 'msgpack-v1'
);
```

This assumes the developer-supplied codec's `name()` returns `msgpack-v1`. Register a handler for `reports.generate.v1`; codecs decode data and handlers execute it. Dispatching a PHP object through `enqueue()` still selects `php`, or `opis-closure-v4` for `ClosureJob`. Use `publish()` to select another encoding explicitly.

## Verify an adapter

Before distributing a new adapter, exercise concurrent claims, delayed availability, expired-lease recovery, stale settlement rejection, attempt preservation, binary-body round trips, backend exceptions, and terminal failure retention. Test optional capabilities only when implemented. Test consumers with both JSON and native PHP bodies so the adapter does not accidentally depend on text-only payloads.
