# Configuration and Database Setup

Queue settings live in the application's `queue_transport` configuration. Ensure `Yuga\Queue\QueueServiceProvider` is loaded through your application's service-provider configuration. It registers the dispatcher, codecs, job discovery, handlers, workers, and queue console commands.

## Minimal database configuration

Create `config/queue_transport.php` using the configuration directory appropriate to your application:

```php
<?php

return [
    'default' => 'database',
    'connections' => [
        'database' => [
            'driver' => 'database',
            'table' => 'queue_messages',
        ],
    ],
    'worker' => [
        'max_attempts' => 3,
        'retry_delay' => 5,
        'reservation_seconds' => 60,
    ],
    'discovery' => [
        'roots' => [path('app/Jobs') => 'App\\Jobs'],
    ],
    'php_allowed_classes' => [],
];
```

Create `app/Jobs` before enabling that discovery root. Root paths must exist, and files must match their Composer PSR-4 class names. Adjust the namespace if your application uses something other than `App`.

Use integer values for worker settings; cast environment values explicitly if necessary. `max_attempts` includes the initial execution. `retry_delay` is measured in seconds. `reservation_seconds` controls the polling worker's lease, not RoadRunner's broker acknowledgement timeout.

The committed provider obtains its PDO from `Yuga\Database\Migration\PDO::getInstance()->getConnection()`. Adding `dsn`, `username`, or `password` keys alone does not switch that binding to a separate connection. Configure the application's database connection, or explicitly replace the binding when isolation is needed. The transport currently supports MySQL.

## Message table

Create this table through an application migration or your database tooling. The schema below matches the fields read and written by `DatabaseTransport`; there is no registered migration-generation command for this new table.

```sql
CREATE TABLE queue_messages (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
    job_id VARCHAR(64) NOT NULL,
    queue VARCHAR(191) NOT NULL,
    type VARCHAR(255) NOT NULL,
    encoding VARCHAR(64) NOT NULL,
    version INT UNSIGNED NOT NULL,
    body LONGBLOB NOT NULL,
    status VARCHAR(16) NOT NULL DEFAULT 'waiting',
    attempts INT UNSIGNED NOT NULL DEFAULT 0,
    available_at BIGINT NOT NULL,
    reservation_token CHAR(64) NULL,
    reserved_until BIGINT NULL,
    failure_reason LONGTEXT NULL,
    created_at BIGINT NOT NULL,
    updated_at BIGINT NOT NULL,
    INDEX queue_available (queue, status, available_at),
    INDEX queue_reserved (queue, status, reserved_until),
    INDEX envelope_id (job_id)
) ENGINE=InnoDB;
```

The body is raw bytes, so use a binary column. Do not convert native PHP or closure payloads into JSON or base64 inside the transport. Timestamps are Unix seconds. Successful rows remain with status `done`; arrange retention separately if needed.

Use autocommit for worker reservation and settlement operations. Do not wrap the worker loop in an application transaction. Publishing within a transaction follows that PDO connection's transaction behaviour; this API does not provide an outbox or automatic after-commit dispatch.

## Failure-store table

RoadRunner terminal failures use a separate `queue.failures` service. Its default binding uses the same application's MySQL PDO and table `queue_failures`:

```sql
CREATE TABLE queue_failures (
    id CHAR(64) NOT NULL PRIMARY KEY,
    source VARCHAR(191) NOT NULL,
    delivery_id VARCHAR(191) NOT NULL,
    queue VARCHAR(191) NOT NULL,
    job_id VARCHAR(64) NOT NULL,
    type VARCHAR(255) NOT NULL,
    encoding VARCHAR(64) NOT NULL,
    version INT UNSIGNED NOT NULL,
    body LONGBLOB NOT NULL,
    attempts INT UNSIGNED NOT NULL,
    reason LONGTEXT NOT NULL,
    failed_at BIGINT NOT NULL,
    replay_status VARCHAR(16) NOT NULL DEFAULT 'pending',
    replay_token CHAR(64) NULL,
    replay_started_at BIGINT NULL,
    replayed_at BIGINT NULL,
    INDEX source_failed (source, failed_at),
    INDEX envelope_id (job_id)
) ENGINE=InnoDB;
```

For an existing failure table, add the four replay columns through a migration instead of recreating the table. The store requires autocommit and rejects recording or replay transitions during an active transaction.

Database polling failures stay in `queue_messages`. They are managed by `queue:failed`, `queue:retry`, and `queue:forget`; the separate failure store is managed by `queue:failures` and `queue:replay`.

## Service-provider extensions

Register custom handlers, jobs, codecs, and drivers from a provider after the queue services are available. Use its `boot()` method for examples in these guides. Do not duplicate an already registered handler type or codec name.

The committed provider includes demonstration handlers such as `notifications.send` and `test.roadrunner.failure`. Replace or remove test registrations when adapting the provider for an application; they are not required parts of the envelope protocol.
