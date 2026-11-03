# Jobs and Dispatching

## Generate and discover a PHP job

```bash
php yuga make:job Reports/GenerateReport
```

This creates `app/Jobs/Reports/GenerateReport.php`. Nested names are supported and an existing file is never overwritten. The command uses `APP_NAMESPACE` (default `App`) for the class namespace.

```php
<?php

namespace App\Jobs\Reports;

use Yuga\Interfaces\Queue\JobInterface;
use Yuga\Interfaces\Queue\ShouldQueueInterface;
use Yuga\Queue\Attributes\QueueJob;
use Yuga\Queue\DispatchableTrait;

#[QueueJob]
class GenerateReport implements JobInterface, ShouldQueueInterface
{
    use DispatchableTrait;

    public function __construct(public int $reportId) {}

    public function run(): void
    {
        // Load the report and generate it using application services.
    }
}
```

`JobInterface` is a marker used by the job registry. `ShouldQueueInterface` tells `dispatch()` to enqueue the object. `#[QueueJob]` opts the class into discovery; it does not select a connection or encoding. Discovery requires a configured root and an autoloadable concrete class with a public instance `run()` method.

The registry supplies the allowed PHP job classes to the PHP codec. Package providers can explicitly register a job without an attribute:

```php
app('queue.jobs')->register(\Vendor\Package\Jobs\RefreshIndex::class);
```

## Dispatch immediately or enqueue

```php
GenerateReport::dispatch(42);     // Enqueue: implements ShouldQueueInterface.
GenerateReport::dispatchNow(42);  // Execute in the current process.

app('queue.dispatcher')->dispatch(new GenerateReport(42));
```

`dispatch()` executes an object synchronously when it does not implement `ShouldQueueInterface`. Use `enqueue()` to explicitly queue an object regardless of that marker:

```php
$envelope = app('queue.dispatcher')->enqueue(
    job: new GenerateReport(42),
    connection: 'database',
    queue: 'reports',
    delaySeconds: 10
);
```

Queued dispatch returns a `JobEnvelope`; synchronous dispatch returns the result of `run()`. Publication success means the backend accepted the publication call, not that execution has completed.

## Fluent dispatch

```php
$envelope = GenerateReport::prepare(42)
    ->onConnection('database')
    ->onQueue('reports')
    ->delay(10)
    ->send();
```

`send()` is required. `PendingDispatch` does not publish from a destructor. After successful publication it cannot be sent or changed again.

For plain `dispatch()`, the dispatcher reads public `$connection`, `$queue`, and `$delay` properties from the job, with defaults of the configured connection, `default`, and zero seconds. `prepare()` instead starts with those defaults and uses the fluent choices; it does not inherit those job properties.

## Serializable data and execution services

Store IDs and data in the constructor. Resolve database connections, HTTP clients, and other runtime services inside `run()` through method injection or application services. Do not serialize open resources or active request state.

A PHP job's envelope type is its fully qualified class name. The consumer must autoload that class, allow it through the job registry, and deploy compatible code. Native PHP jobs use `serialize()` rather than base64 encoding.

For nested DTO objects, list the required additional classes in `php_allowed_classes`. Allowing a job class does not automatically allow every nested class. An allowlist is not a sandbox: serialized PHP objects are only appropriate for trusted producers.

## Optional polling context

```php
use Yuga\Queue\JobContext;

public function run(?JobContext $context = null): void
{
    // Work in bounded chunks. Before the lease expires, if supported:
    $context?->renew(30);
}
```

The execution bridge supplies a parameter named `context` when present. Polling workers supply `JobContext`; the current RoadRunner consumer supplies `null`. Keep it nullable for jobs shared across both. Renewal adds seconds to an owned, unexpired lease; it is cooperative, not an automatic heartbeat.

## Closures

```php
use Yuga\Queue\Jobs\ClosureJob;

$prefix = 'Report';

ClosureJob::prepare(
    callback: static function (array $data) use ($prefix): void {
        error_log($prefix . ': ' . $data['id']);
    },
    data: ['id' => 42]
)->onConnection('database')->onQueue('reports')->send();
```

Closures use type `yuga.closure.v1`, encoding `opis-closure-v4`, and Opis Closure `^4.5`. Define serializable closures in application PHP files and capture only the data they need. The whole `ClosureJob`, including its optional failure callback, is serialized. This format is for trusted PHP applications with compatible code and dependencies; use JSON for cross-language jobs.

## Terminal failure callback

A PHP job can implement `Yuga\Queue\CanFailInterface`:

```php
public function onFailure($job, $message)
{
    error_log("Job {$job->id} failed after {$job->attempts} attempts: {$message}");
}
```

The callback receives a context object with `id`, `attempts`, and `type`; it is not the original database row. A `ClosureJob` accepts `failureCallback: static function ($job, $message) { ... }` for the same purpose.

Callbacks run after terminal settlement and are best effort. Their exceptions emit `queue:callback-failure` without retrying the original job. If decoding fails, there may be no decoded object whose callback can run.
