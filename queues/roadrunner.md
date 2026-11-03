# RoadRunner

RoadRunner delivers tasks to long-lived Jobs workers. Yuga's `RoadRunnerPublisher` implements publication; `RoadRunnerConsumer` maps received tasks into envelopes and uses the shared job executor. It is not a polling transport for `queue:work`.

## Requirements

Install `spiral/roadrunner-jobs` in the application and `spiral/roadrunner-http` when also serving HTTP. Use mutually compatible SDK and RoadRunner binary versions and retain the resolved versions in `composer.lock`.

The integration was exercised with a RoadRunner binary reporting `3.0.0` and `spiral/roadrunner-jobs` `4.x-dev` at commit `5876823ac628fd72e4586a98fed88af01f9bc49a`. That SDK uses the `Spiral\RoadRunner\Jobs` namespace while several underlying Composer packages use the `roadrunner/*` names. These are tested versions, not a blanket compatibility promise for every release. `yuga/runtime` supplies `RoadRunnerServer`; its launcher uses Symfony Process, which the framework currently requires.

Download/build a binary for the host OS and CPU architecture with the Jobs plugin and desired backend driver. Confirm it runs:

```bash
./rr --version
```

An `exec format error` indicates an executable/host mismatch or invalid file, before PHP queue code runs.

## Queue connection

Add this connection to `queue_transport`:

```php
'roadrunner' => [
    'driver' => 'roadrunner',
    'rpc' => 'tcp://127.0.0.1:6003',
    'failure_source' => 'fntyuga-roadrunner-test',
    'pipelines' => [
        'default' => 'yuga-test',
        'notifications' => 'yuga-test',
        'yuga-test' => 'yuga-test', // Actual destination used during replay.
    ],
    'worker' => [
        'max_attempts' => 3,
        'retry_delay' => 5,
    ],
],
```

The outer key is a connection name; `driver` selects the adapter. Logical queues map to RoadRunner pipeline names. Mapping a name does not create the pipeline in RoadRunner. Set the top-level `default` to `roadrunner` if normal dispatch should use it.

Connection worker settings replace the global worker array when present. Prepare the MySQL failure store even when normal jobs use RoadRunner; the default consumer resolves `queue.failures` at startup. Configure the source's [replayer](failures.md) as well.

## Queue-only profile

Create `.rr.queue.yaml` in the application root:

```yaml
version: "3"

server:
  command: "php yuga runtime:worker --queue-connection=roadrunner"
  relay: pipes

rpc:
  listen: tcp://127.0.0.1:6003

jobs:
  consume: [yuga-test]
  pool:
    num_workers: 1
  pipelines:
    yuga-test:
      driver: memory
      config:
        priority: 10
        prefetch: 10
```

The memory driver is useful for local checks but loses queued work when the server stops. Select and configure an appropriate durable RoadRunner backend for deployments that require persistence.

## Combined HTTP and queue profile

Create `.rr.combined.yaml` using the queue-only configuration above and add:

```yaml
http:
  address: 127.0.0.1:8000
  middleware: [static]
  pool:
    num_workers: 2

static:
  dir: public
  forbid: [".php", ".htaccess", ".env"]
```

This starts separate HTTP and Jobs pools in one RoadRunner process. RoadRunner sets `RR_MODE`; `runtime:worker` selects the HTTP adapter or queue consumer accordingly. Keep PHP diagnostics and direct writes away from STDOUT because the worker relay uses it. The queue consumer redirects buffered application output to STDERR; direct `fwrite(STDOUT, ...)` bypasses that buffering.

## Launcher configuration

Set `config/runtime_server.php`:

```php
<?php

return [
    'http_runtime' => 'roadrunner',
    'roadrunner' => [
        'binary' => path(PHP_OS_FAMILY === 'Windows' ? 'rr.exe' : 'rr'),
        'http_config' => path('.rr.http.yaml'),
        'combined_configs' => [
            'roadrunner' => path('.rr.combined.yaml'),
        ],
        'queue_configs' => [
            'roadrunner' => path('.rr.queue.yaml'),
        ],
        'queue_connection' => 'roadrunner',
    ],
];
```

```bash
# One process starts HTTP and Jobs:
php yuga start

# Alternative: Jobs only, with HTTP served elsewhere:
php yuga queue:serve roadrunner
```

Stop the first profile before starting the second if their RPC ports overlap. `queue:serve` currently supports RoadRunner connections only. Its argument is required.

For RoadRunner HTTP without Jobs, create `.rr.http.yaml` with `server`, `rpc`, `http`, and `static` from the combined example but omit `jobs`. Set `queue_connection` to `null` to select it. If that key is omitted, `start` inspects the default queue connection; a RoadRunner driver selects its combined profile. With `'http_runtime' => 'php'`, `start` launches PHP's development server and does not start a RoadRunner queue service.

`--host` and `--port` on `start` apply to the PHP server path. RoadRunner's address comes from its selected YAML profile. The launcher does not rewrite it.

For direct operation, the equivalent process startup is:

```bash
./rr serve -c .rr.combined.yaml
```

`runtime:worker` and `queue:consume-roadrunner` are internal entry points managed by RoadRunner. Running the latter directly in a normal terminal is rejected because `RR_MODE` is not `jobs`.

## Dispatch and verify

```php
App\Jobs\Reports\GenerateReport::prepare(42)
    ->onConnection('roadrunner')
    ->onQueue('default')
    ->send();
```

PHP objects, JSON messages, and `ClosureJob` use the same publisher. Register JSON handlers and PHP discovery before starting workers. Restart/recycle workers after deploying code or changing registrations so they load the new application state.

For a failure test with maximum attempts 3 and delay 5, expect `queue:retrying` after attempts 1 and 2, then `queue:failure` on attempt 3. A terminal `jobs nack request` log is expected for an intentionally failed job. Callback failure is reported separately. The failure store is written before terminal nack; execution callbacks run afterward.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| Unable to connect to `127.0.0.1:6003` | Server is running and RPC addresses agree; on macOS use `lsof -nP -iTCP:6003 -sTCP:LISTEN` |
| `no such pipeline` | YAML defines the pipeline and the correct profile was started |
| No mapping for a logical queue | Add that queue to the connection's `pipelines` array |
| Unknown PHP job class or incomplete object | Autoload, discovery roots, class attribute, and nested DTO allowlist |
| Unknown named parameter `context` | Use the current execution bridge; it supplies context only when `run()` declares that parameter |
| Worker EOF after an HTTP response helper | Remove process-terminating response paths; return a PSR-7 response |
| Empty/incorrect static asset response | Verify files under `public` and framework 404 propagation; see [runtimes](../runtimes.md) |

RoadRunner reservation renewal/context and selective queue-pool restart through `queue:restart` remain unimplemented. The attempt header tracks explicit retries, not every broker crash redelivery. Backend-specific acknowledgements, persistence, and redelivery semantics must be verified for the selected RoadRunner driver.
