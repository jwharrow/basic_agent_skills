# app-observability

A Factory plugin that runs a production health check for any project across Firebase Crashlytics and Datadog. Configure once per project; reuse forever.

## What it does

Investigates production health across eight phases and produces a structured summary:

- **Phase 0** — Firebase Crashlytics: top crashes, new/regressed issues, impacted users, device and version concentration
- **Phase 1** — Datadog incidents (active, stable, recently resolved)
- **Phase 2** — Alerting monitors
- **Phase 3** — RUM errors (per mobile/web app)
- **Phase 4** — RUM performance (view loading times, action error rates)
- **Phase 5** — Log errors (clustered by pattern)
- **Phase 6** — APM span health (per-resource error rate and P95 latency)
- **Phase 7** — Downstream service dependencies

All read-only. The only thing it writes is the per-project config file.

## Prerequisites

1. **Firebase MCP**

   ```bash
   droid mcp add firebase -- npx -y firebase-tools@latest experimental:mcp
   ```

   Then authenticate:

   ```bash
   firebase login
   ```

   The logged-in account must have access to the Firebase projects referenced in your config.

2. **Datadog MCP** — connect via Factory's MCP settings.

## Install

```bash
droid plugin marketplace add https://github.com/jwharrow/basic_agent_skills
droid plugin install app-observability@basic-agent-skills
```

## Usage

From any project directory:

```
/health-check
```

Or ask Droid conversationally: "run a health check", "any crashes in prod?", "check production health".

### Arguments

- `/health-check reconfigure` — rerun the interactive setup, overwriting the saved config
- `/health-check app=<name>` — scope the investigation to a single configured app
- `/health-check window=now-4h` — override the default time window

## First run

On first invocation in a project, the skill walks you through a short interactive setup and writes the answers to `.factory/app-observability.json`. You'll be asked:

1. Which apps to monitor (name + platform: `ios`, `android`, `web`, `backend`)
2. Firebase project ID and app ID for each mobile/web app
3. Datadog `service` tag for each app
4. Any additional backend services (optional)
5. Datadog env (default `prod`), site (default `US1`), window (default `now-24h`)

After writing the file, the skill offers to stage it for commit so teammates can share the config.

## Reconfigure

```
/health-check reconfigure
```

Overwrites the saved config and runs setup again.

## Config schema

`.factory/app-observability.json`:

```json
{
  "apps": [
    {
      "name": "my_app_ios",
      "platform": "ios",
      "datadog_service": "my_app_ios",
      "firebase": {
        "project_id": "my-firebase-project",
        "app_id": "1:123456789:ios:abcdef0123456789"
      }
    }
  ],
  "backend_services": [
    {
      "name": "my-api",
      "datadog_service": "my-api"
    }
  ],
  "datadog": {
    "env": "prod",
    "site": "US1"
  },
  "window": "now-24h"
}
```

| Field | Required | Notes |
|-------|----------|-------|
| `apps[].name` | yes | Human-readable identifier |
| `apps[].platform` | yes | One of `ios`, `android`, `web`, `backend` |
| `apps[].datadog_service` | yes | The `service` tag used in Datadog queries |
| `apps[].firebase.project_id` | mobile/web only | Firebase project ID |
| `apps[].firebase.app_id` | mobile/web only | Format: `1:NUMBER:PLATFORM:HASH` |
| `backend_services[]` | no | Additional services for APM/log coverage |
| `datadog.env` | no | Default: `prod` |
| `datadog.site` | no | Default: `US1` |
| `window` | no | Default: `now-24h` |

## Examples

### Mobile + backend (full stack)

```json
{
  "apps": [
    {
      "name": "dealer_app_ios",
      "platform": "ios",
      "datadog_service": "dealer_app_ios",
      "firebase": {
        "project_id": "inc-carscommerce-app",
        "app_id": "1:857017453616:ios:f4c9ecfc140d6d564d71f5"
      }
    }
  ],
  "backend_services": [
    { "name": "dealer_graph", "datadog_service": "dealer_graph" }
  ],
  "datadog": { "env": "prod", "site": "US1" },
  "window": "now-24h"
}
```

### Backend only (no Crashlytics)

```json
{
  "backend_services": [
    { "name": "my-api", "datadog_service": "my-api" },
    { "name": "my-worker", "datadog_service": "my-worker" }
  ],
  "datadog": { "env": "prod", "site": "US1" },
  "window": "now-24h"
}
```

Phase 0 (Crashlytics) is skipped when no `apps` are configured.

### Mobile only (no backend)

```json
{
  "apps": [
    {
      "name": "my_app_android",
      "platform": "android",
      "datadog_service": "my_app_android",
      "firebase": {
        "project_id": "my-project",
        "app_id": "1:123:android:abc"
      }
    }
  ],
  "datadog": { "env": "prod", "site": "US1" },
  "window": "now-24h"
}
```

## Output

The skill returns a structured dashboard followed by plain-text details:

- `<json-render>` status overview with a `StatusLine` per phase and high-level metrics
- Plain-text findings with specific numbers, service names, error messages, and deep links back to Firebase and Datadog consoles
- Recommended next steps for each issue

If everything is clean, it says so explicitly.
