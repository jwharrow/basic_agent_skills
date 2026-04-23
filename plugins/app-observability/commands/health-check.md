---
description: Run a production health check across Firebase Crashlytics and Datadog for this project
---

Invoke the `health-check` skill.

Forward any user arguments verbatim as `$ARGUMENTS`. Recognized hints:

- `reconfigure` — rerun the interactive setup
- `app=<name>` — scope to a single configured app
- `window=<relative>` — override the default time window (e.g. `window=now-4h`)

If `$ARGUMENTS` is empty, run a normal health check using the saved `.factory/app-observability.json` config, or trigger setup if no config exists.
