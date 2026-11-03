---
description: Yuga commands
---

# Commands

## Application and queue commands

| Command | Purpose |
| --- | --- |
| `start` | Start the configured HTTP server, optionally with RoadRunner Jobs |
| `make:job` | Generate a discoverable queueable PHP job |
| `queue:work` | Poll a connection implementing the transport contract |
| `queue:serve` | Start a RoadRunner queue-only server profile |
| `queue:consume-roadrunner` | Internal RoadRunner Jobs consumer entry point |
| `runtime:worker` | Internal RoadRunner HTTP/Jobs mode selector |
| `queue:restart` | Ask polling workers to exit after current work |
| `queue:monitor` | Snapshot a connection declaring monitoring support |
| `queue:failed`, `queue:retry`, `queue:forget` | Manage transport-retained failures |
| `queue:failures`, `queue:replay` | Inspect and replay the separate failure store |
| `queue:roadrunner-test` | Integration-test command; not the normal production consumer |

See [Queues](../queue-worker.md) and [Application Runtimes](../runtimes.md) for setup, capability limits, and examples. Use `php yuga help COMMAND` for the installed version's arguments and options.

