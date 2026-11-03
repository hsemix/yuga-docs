# Helpers

## Asset URLs

`asset()` and `assets()` generate the same URL:

```php
asset('assets/css/app.css');
assets('yuga/js/ys.min.js');
```

With `ASSET_URL` empty, these produce `/assets/css/app.css` and `/yuga/js/ys.min.js`. An optional `ASSET_URL` supplies a subdirectory or CDN prefix. Explicit HTTP(S) and protocol-relative URLs are preserved.

These helpers generate URLs; they do not copy assets into `public`. See [static asset setup](runtimes.md#static-assets) for RoadRunner.

## Queue dispatcher

Use `app('queue.dispatcher')` for explicit dispatch and publication, or a job's `DispatchableTrait` methods for class-based dispatch. See [Jobs and Dispatching](queues/jobs.md) and [Payloads and External Consumers](queues/payloads.md).

