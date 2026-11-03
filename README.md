---
description: A php framework for people with mvvm background knowledge,
---

# Yuga Framework

To get started you just need one command `composer create-project yuga/yuga your-project-folder`

Yuga has two programming paradigms it uses i.e mvvm and mvc, the developer gets to decide which paradigm they want to use but by default the framework comes set with mvvm structure.\
If you want to change this setting, you go to `environment/.env` file and locate the paradigm key, change it to mvc and you are done.

**Local Development Server**

Use the `start` Yuga command to launch the configured HTTP server. With `runtime_server.http_runtime` set to `php` (the default), it starts PHP's development server at `http://localhost:8000`:

```bash
php yuga start
```

With RoadRunner configured, the same command can start HTTP and queue workers together. See [Application Runtimes](runtimes.md) for profile selection and [Queues](queue-worker.md) for database workers, payload formats, RoadRunner, and custom adapters.
