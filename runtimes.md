# Application Runtimes

`yuga/runtime` owns HTTP runtime adapters and the RoadRunner process launcher. The framework owns Yuga console commands and application/queue integration. The application owns server profiles and connection configuration.

## Start the selected runtime

```bash
php yuga start
```

`runtime_server.http_runtime` selects the startup path:

| Value | Startup |
| --- | --- |
| `php` | PHP built-in development server; `--host` and `--port` apply |
| `roadrunner` | Selected RoadRunner HTTP-only or combined YAML profile |

Other adapter classes do not automatically imply support in the `start` launcher. The current command rejects other runtime names. FPM is managed by the web server/process manager, not this development command.

For RoadRunner, `roadrunner.queue_connection` selects a queue connection. A RoadRunner queue driver chooses the named `combined_configs` entry; otherwise the launcher selects `http_config`. Explicit `null` means HTTP-only. If omitted, the default queue connection is inspected. See the complete [RoadRunner setup](queues/roadrunner.md).

```bash
# Jobs only; useful with HTTP handled by FPM or another server:
php yuga queue:serve roadrunner
```

This selects `queue_configs[connection]`. The launcher starts one foreground `rr serve -c ...` process without a Symfony Process execution timeout. It does not start a second server from an HTTP request or from dispatch. Profiles sharing an RPC address are alternatives, not simultaneous processes.

## Worker entry point

RoadRunner profiles use:

```yaml
server:
  command: "php yuga runtime:worker --queue-connection=roadrunner"
  relay: pipes
```

RoadRunner's `RR_MODE` determines whether that entry point runs `RoadRunnerRuntime` for HTTP or calls `queue:consume-roadrunner` for Jobs. It is an internal managed worker command. An HTTP pool and a Jobs pool have separate PHP processes even though the same command starts them.

## Return HTTP responses

HTTP application code must return its response through the framework/runtime bridge. For example:

```php
return response()->json(['message' => 'Created'], code: 201);
```

The current JSON helper returns a PSR-7 response instead of echoing and exiting. `jsonResponse()` delegates to it. Keep `return` at existing call sites; execution now continues if a caller discards the response. The argument order remains `value`, `options`, `code`, `dept`, with an array or `JsonSerializable` value.

Avoid `exit()` and `die()` in request handling and response helpers. They terminate a persistent worker before RoadRunner receives its protocol response, which can appear as worker EOF and HTTP 500. A missing route must propagate a returned 404 response through controller, route rendering, and exception handling. GET should return its error body; HEAD should return the status/headers without a body.

The current `Request::isAjax()` also recognises curl/wget user agents. Thus a curl request to a missing route may receive a JSON 404 while a browser receives HTML. This is existing request classification, not a RoadRunner-specific content-negotiation rule.

## Static assets

Set RoadRunner's static middleware to the application's public directory:

```yaml
http:
  address: 127.0.0.1:8000
  middleware: [static]

static:
  dir: public
  forbid: [".php", ".htaccess", ".env"]
```

`/assets/css/app.css` maps to `public/assets/css/app.css`. URL generation does not publish or create a file. Missing static files can fall through to the application, which must return 404 rather than an empty 200 or worker failure. Remove references to decommissioned assets; do not restore legacy scripts solely to silence their missing-file requests.

The updated `asset()` and `assets()` helpers use root-relative URLs by default:

```php
asset('assets/css/app.css'); // /assets/css/app.css
```

`ASSET_URL` optionally supplies a base prefix, such as `/myapp` or `https://cdn.example.com`. Leave it empty for assets served from the current origin root. It does not change RoadRunner's static directory or repair missing files. There is no need to hardcode an application host merely to generate root-relative assets.

## Verify response handling

Use the port in the selected YAML profile:

```bash
curl -I http://127.0.0.1:8000/assets/css/app.css
curl -i http://127.0.0.1:8000/missing-test.js
curl -I http://127.0.0.1:8000/missing-test.js
curl -i http://127.0.0.1:8000/
```

Expect CSS 200 with a CSS content type, missing GET/HEAD 404, and homepage 200. Repeat requests to catch persistent-worker problems. A queue-only profile should not bind the HTTP port; on macOS check `lsof -nP -iTCP:8000 -sTCP:LISTEN` with other HTTP servers stopped.
