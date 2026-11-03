# Payloads and External Consumers

## Envelope

`Yuga\Queue\Payload\JobEnvelope` carries:

| Field | Meaning |
| --- | --- |
| `version` | Envelope protocol version, currently `1` |
| `id` | Portable job identity; newly created envelopes use 32 hexadecimal characters |
| `type` | Handler name, or PHP job class name |
| `encoding` | Registered codec name |
| `body` | Encoded payload as raw bytes |

Transport adapters preserve the envelope ID and body. A backend delivery ID is separate: RoadRunner supplies a task ID, and the database supplies a row ID. `headers()` returns envelope metadata without the body. There is no requirement that every transport wrap the whole envelope in JSON.

## Publish JSON

```php
$envelope = app('queue.dispatcher')->publish(
    type: 'reports.generate.v1',
    payload: ['report_id' => 42],
    connection: 'database',
    queue: 'reports'
);
```

`publish()` defaults to `json`. The JSON codec accepts JSON-compatible arrays, scalars, and null, and reports encoding/decoding errors. It does not serialize PHP job objects. The same type can be consumed by a non-PHP worker that understands your payload schema.

Register a matching handler in an application provider's `boot()`:

```php
use Yuga\Queue\Payload\JobEnvelope;

app('queue.handlers')->register(
    'reports.generate.v1',
    static function (mixed $data, JobEnvelope $envelope): void {
        if (!is_array($data) || !isset($data['report_id']) || !is_int($data['report_id'])) {
            throw new \InvalidArgumentException('An integer report_id is required.');
        }

        // Generate the report, using $envelope->id for correlation.
    }
);
```

Handlers receive decoded payload, envelope, and an optional nullable `JobContext` as their third argument. Payload validation belongs to the handler. Explicit handlers take priority over the PHP-object fallback and can be used with other registered encodings.

Version your application message types or schemas independently of the envelope version. Changing `reports.generate.v1` does not require changing the transport protocol.

## PHP objects and closures

| Encoding | Body | Execution |
| --- | --- | --- |
| `json` | JSON text | Explicit handler registered for the type |
| `php` | Native PHP `serialize()` bytes | Explicit handler, or matching job object's public `run()` |
| `opis-closure-v4` | Opis-serialized `ClosureJob` | Built-in `yuga.closure.v1` handler |

The PHP fallback requires the decoded object's exact class name to equal the envelope type. PHP and closure formats require trusted publishers and compatible PHP code. Do not deserialize arbitrary messages from unauthenticated external producers. Opis closure decoding does not use the native PHP codec's job-class allowlist.

## RoadRunner wire mapping

RoadRunner task name is the envelope type. Task payload is exactly the envelope body. Yuga adds these headers, each with one string value:

| Header | Example | Meaning |
| --- | --- | --- |
| `yuga-version` | `1` | Envelope version |
| `yuga-id` | `a1ac6b7ee82cb299cd9db28f962e939d` | Portable job ID |
| `yuga-type` | `reports.generate.v1` | Must match the task name |
| `yuga-encoding` | `json` | Body decoder |
| `yuga-queue` | `reports` | Logical Yuga queue selected by the publisher |
| `yuga-attempt` | `1` | Explicit execution attempt counter |

An external JSON consumer reads these headers, decodes the task payload as JSON, selects a handler by task name, then acknowledges or applies its own failure policy. It does not need to deserialize a PHP class. An external producer targeting Yuga must provide the required metadata and matching task name. Missing `yuga-attempt` is accepted as attempt 1 for older messages; other required envelope headers are validated strictly.

The actual pipeline is obtained from `getPipeline()`. `getQueue()` is broker-specific and may not identify the pipeline. Native PHP and Opis bodies may contain binary bytes; retain them exactly.

The attempt header counts explicit Yuga requeues. It is not a durable count of all broker redeliveries after worker crashes.
