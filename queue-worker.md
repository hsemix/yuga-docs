---
description: Dispatch jobs, select payload formats, and run database or RoadRunner consumers.
---

# Queues

Yuga separates job execution, payload encoding, and delivery. The same dispatcher can send a JSON message, a PHP job object, or a serialized closure through a configured connection.

These pages describe the current queue API in the framework source. Older installations may still contain the previous queue implementation; update the framework and application configuration together before using these examples.

The API examples were checked against [framework revision `6e45e01`](https://github.com/hsemix/yuga-framework/tree/6e45e01cea664f7627917ee4b0b39d928cf5897d/src/Yuga/Queue). A Composer release installed in an application may lag behind that source revision.

## Choose a connection and payload

| Concern | Available implementation | Use |
| --- | --- | --- |
| Delivery | MySQL database transport | Poll jobs with `queue:work` |
| Delivery | RoadRunner publisher and consumer | Run Jobs workers managed by RoadRunner |
| Payload | `json` | Named messages that external consumers can process |
| Payload | `php` | Native PHP serialized job objects |
| Payload | `opis-closure-v4` | PHP closures and captured data using Opis 4.5-compatible serialization |

JSON is a payload format, not a queue driver. Redis and other backends can be added through the [adapter contracts](queues/extending.md); they are not built-in drivers in this API.

## First database job

Complete [configuration and database setup](queues/configuration.md), then generate a job:

```bash
php yuga make:job Reports/GenerateReport
```

Add serializable constructor data and a public `run()` method to the generated class. Dispatch it and start a consumer:

```php
App\Jobs\Reports\GenerateReport::dispatch();
```

```bash
php yuga queue:work database --queue=default
```

The generated job implements `ShouldQueueInterface`, so `dispatch()` enqueues it. Objects without that marker execute synchronously through the dispatcher.

## RoadRunner startup

Once [RoadRunner configuration](queues/roadrunner.md) is in place:

```bash
# HTTP and queues together, when selected in runtime_server configuration:
php yuga start

# Alternatively, queues only when another server handles HTTP:
php yuga queue:serve roadrunner
```

Run these as alternative profiles when they share an RPC port. Publishing requires the selected backend to be available; dispatching does not automatically start RoadRunner.

## Explore the API

- [Jobs and dispatching](queues/jobs.md): attributes, discovery, delayed jobs, and closures.
- [Payloads and external consumers](queues/payloads.md): envelope metadata and wire formats.
- [Workers and lifecycle](queues/workers.md): retries, reservations, listeners, and restart behaviour.
- [Failures and replay](queues/failures.md): inspect and recover terminal failures.
- [Custom adapters and codecs](queues/extending.md): add another backend or encoding.

Jobs should tolerate repeated delivery. An envelope ID identifies a job; it does not by itself prevent duplicate business effects. Store an application-level idempotency key when repeated execution would be harmful.

