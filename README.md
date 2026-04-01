# Console Profiler Bundle

[![CI](https://github.com/rcsofttech85/ConsoleProfilerBundle/actions/workflows/ci.yaml/badge.svg)](https://github.com/rcsofttech85/ConsoleProfilerBundle/actions/workflows/ci.yaml)
[![Version](https://img.shields.io/packagist/v/rcsofttech/console-profiler-bundle.svg?label=stable)](https://packagist.org/packages/rcsofttech/console-profiler-bundle)
[![Codacy Badge](https://app.codacy.com/project/badge/Grade/828f3e302ce84185a0b0befdac5f1b27)](https://app.codacy.com/gh/rcsofttech85/ConsoleProfilerBundle/dashboard?utm_source=gh&utm_medium=referral&utm_content=&utm_campaign=Badge_grade)
[![Codacy Badge](https://app.codacy.com/project/badge/Coverage/828f3e302ce84185a0b0befdac5f1b27)](https://app.codacy.com/gh/rcsofttech85/ConsoleProfilerBundle/dashboard?utm_source=gh&utm_medium=referral&utm_content=&utm_campaign=Badge_coverage)

If you've ever watched a long-running Symfony console command crawl and
wondered, "Is this thing leaking memory? Am I hammering the database with N+1
queries right now?" — this bundle is for you.  

Read more about it on Medium at https://medium.com/@rcsofttech85/from-personal-tool-to-open-source-console-profiler-bundle-12d0bac0f758.

The standard Symfony Profiler is amazing for HTTP requests, but it doesn't help
you much when a queue worker is eating up RAM in the background. The Console
Profiler Bundle hooks right into your terminal to give you a live, **premium TUI
dashboard** while your commands are actually running.

![Console Profiler Dashboard](docs/dashboard.png)

---

## Features

* Live, auto-refreshing TUI dashboard pinned to the top of your terminal
* Memory usage, peak memory, and growth rate with color-coded bars
* Real-time trend indicators (`↑` `↓` `→`) for memory and SQL
* CPU user/system time tracking via `getrusage()`
* Automatic SQL query counting via Doctrine DBAL 4 Middleware
* JSON profile export for CI pipeline regression testing
* Exit code stamping on command completion
* Zero configuration required — works out of the box
* Graceful degradation without `ext-pcntl` (no auto-refresh)

---

## Installation

Pop it into your dev dependencies via Composer:

```bash
composer require --dev rcsofttech/console-profiler-bundle
```

*Note: You'll need PHP 8.4+, Symfony 8.0+, and the `ext-pcntl` extension
(which you probably already have on Mac/Linux) to get the smooth async UI
updates.*

---

## Configuration (Optional)

You don't have to configure anything, it works right out of the box. But if you
want to tweak things, create `config/packages/console_profiler.yaml`:

```yaml
console_profiler:
    # Disable the profiler entirely if you want
    enabled: true

    # It's smart enough to turn itself off when kernel.debug is false
    exclude_in_prod: true

    # How often the TUI updates (in seconds)
    refresh_interval: 1

    # Don't bother profiling these noisy default commands
    # (these four are the defaults — add your own as needed)
    excluded_commands:
        - 'list'
        - 'help'
        - 'completion'
        - '_complete'

    # Set this to a path to save a JSON dump for CI regression testing
    profile_dump_path: '%kernel.project_dir%/var/log/profiler/last_run.json'
```

---

## Practical Examples

### 1. Debugging a leaky queue worker

Run your worker normally:

```bash
bin/console messenger:consume async
```

Look at the **Memory** row in the profiler. You'll see a `+X MB/s` indicator
showing exactly how fast memory is growing. If it holds steady into the yellow
or red, you know you've got a leak to fix.

### 2. Guarding against N+1 queries in CI

Set your `profile_dump_path` in `console_profiler.yaml`. Then, in your CI run:

```bash
# Run your heavy sync command
bin/console app:nightly-sync

# Check if someone blew up the query count using jq
SQL_COUNT=$(jq '.counters.sql_queries' var/log/profiler/last_run.json)

if [ "$SQL_COUNT" -gt 500 ]; then
  echo "Whoops! Regression: SQL queries exceeded 500 (got $SQL_COUNT)"
  exit 1
fi
```

The JSON dump tracks memory, CPU times, SQL counts, and more.

---

## JSON Profile Schema

When `profile_dump_path` is configured, the following JSON is written
on command completion:

```json
{
  "timestamp": "2024-01-15T10:30:00+00:00",
  "command": "app:import-data",
  "environment": "dev",
  "exit_code": 0,
  "duration_seconds": 12.4523,
  "memory": {
    "usage_bytes": 16777216,
    "peak_bytes": 33554432,
    "limit_bytes": 268435456,
    "growth_rate_bytes_per_sec": 524288.0
  },
  "cpu": {
    "user_seconds": 8.12,
    "system_seconds": 0.34
  },
  "counters": {
    "sql_queries": 142,
    "loaded_classes": 312,
    "declared_functions": 1204,
    "included_files": 89,
    "gc_cycles": 2
  },
  "system": {
    "php_version": "8.4.12",
    "sapi": "cli",
    "pid": 12345,
    "opcache_enabled": true,
    "xdebug_enabled": false
  }
}
```

---

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-feature`)
3. Ensure tests pass: `vendor/bin/phpunit`
4. Ensure static analysis passes: `vendor/bin/phpstan analyze`
5. Submit a pull request

## License

MIT License.
