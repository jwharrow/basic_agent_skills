---
name: health-check
description: >-
  Runs a production health check for the current project across Firebase Crashlytics
  and Datadog. Covers crashes, incidents, alerting monitors, RUM errors and performance,
  log errors, APM spans, and downstream service health. Uses a saved per-project config
  if present, otherwise prompts for a one-time interactive setup. Use when the user says
  "run a health check", "check production health", "how is the app doing", "any crashes?",
  or invokes `/health-check`.
disable-model-invocation: false
---

# App Health Check

## Input

The user may invoke this skill with optional arguments:

- `reconfigure` — rerun the interactive setup, overwriting the saved config
- `app=<name>` — scope the investigation to a single configured app
- `window=<relative>` — override the default time window (e.g. `window=now-4h`)

If `$ARGUMENTS` is provided, parse these hints from it.

---

## Prerequisites

This skill depends on two MCP servers:

1. **Firebase MCP** — for Crashlytics queries.
   - Install: `droid mcp add firebase -- npx -y firebase-tools@latest experimental:mcp`
   - Auth: user must run `firebase login` and have access to the configured Firebase project
2. **Datadog MCP** — for incidents, monitors, RUM, logs, APM, service dependencies.
   - Connect via Factory integrations settings

If either MCP is unavailable or returns an auth error, stop and tell the user which one needs attention. For Firebase auth errors, specifically say: "Please run `firebase login` and try again."

---

## Configuration

### Locate config

Look for `.factory/app-observability.json` in the current working directory.

- If the file exists and the user did **not** pass `reconfigure`: load it and skip to **Investigation**.
- If the file is missing, or `reconfigure` was passed: run **Interactive Setup**.

### Interactive Setup

Use the `AskUser` tool to gather configuration. Collect answers into a JSON object as you go. Ask in this order:

1. **Apps to monitor**
   Ask: "Which apps should I monitor? List each as `<name>:<platform>` separated by commas (platforms: `ios`, `android`, `web`, `backend`). Example: `dealer_app_ios:ios, dealer_app_android:android, dealer_graph:backend`."

2. **Per-app Firebase details** (only for `ios`, `android`, `web` platforms)
   For each mobile/web app, ask for:
   - Firebase project ID (suggest: "You can find this in the Firebase console URL or in `GoogleService-Info.plist` / `google-services.json`")
   - Firebase app ID (the `1:PROJECTNUMBER:PLATFORM:HASH` format)

3. **Per-app Datadog service name**
   For each app, ask: "What is the Datadog `service` tag for `<name>`?" (default: use the app name)

4. **Backend services** (separate from `backend` platform apps; e.g. downstream services you want APM/log coverage for)
   Ask: "Any additional backend services to include? List as `<name>:<datadog_service>` comma-separated, or press enter to skip."

5. **Environment defaults**
   - Datadog env (default `prod`)
   - Datadog site (default `US1`)
   - Time window (default `now-24h`)

### Write config

Write the collected JSON to `.factory/app-observability.json`. Create the `.factory/` directory if it does not exist.

**Schema:**

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
    {
      "name": "dealer_graph",
      "datadog_service": "dealer_graph"
    }
  ],
  "datadog": {
    "env": "prod",
    "site": "US1"
  },
  "window": "now-24h"
}
```

Omit `firebase` for `backend` platform apps. Omit `apps` or `backend_services` entirely if empty.

### Offer to commit

After writing the config, ask: "Should I commit `.factory/app-observability.json` so teammates can use this config? (yes / no / already-gitignored)"

- If yes: run `git add .factory/app-observability.json` and suggest a commit message like `chore: add app-observability config`. Do **not** push; let the user decide.
- If no: leave the file uncommitted and mention they can commit later.
- If already-gitignored: note it and move on.

---

## Investigation Playbook

Run through each phase in order. For each phase, start broad (aggregates/counts), then drill into anomalies. Skip detailed drill-down if a phase is clean — just mark it OK.

Use `config.window` as the default time window. Parameterize every query with values from the loaded config.

### Phase 0 — Crashlytics Crashes

Skip this phase if `config.apps` contains no entries with a `firebase.app_id`.

For each mobile/web app in `config.apps`:

1. `crashlytics_get_report` with `report: topIssues`, `appId: <firebase.app_id>`. Use a 7-day window for trend context. Note title, errorType, eventsCount, impactedUsersCount, firstSeenVersion, lastSeenVersion, signals.
2. `crashlytics_get_report` with `report: topVersions` — flag version concentration.
3. For any issue with signal `FRESH` or `REGRESSED`, or >5 impacted users:
   - `crashlytics_get_issue` for details
   - `crashlytics_list_events` (pageSize: 2) for sample stack traces
   - Record exception type, blame frame, breadcrumbs, Firebase console URL
4. `crashlytics_get_report` with `report: topAppleDevices` or `topAndroidDevices` — flag hardware concentration.

### Phase 1 — Active Datadog Incidents

Scope to configured services where possible.

- `search_datadog_incidents` with `query: state:(active OR stable)`
- Also `state:resolved` with `from: now-24h`
- Note severity, title, impact.

### Phase 2 — Alerting Monitors

- `search_datadog_monitors` with `status:(alert OR warn)`, filtered by env and any configured service names.
- For each alerting monitor, note name, current value, threshold.

### Phase 3 — RUM Errors

For each mobile/web app in `config.apps`:

- `aggregate_rum_events` with `query: @type:error service:<datadog_service> env:<env>`
- Compute: COUNT(*), group by `@error.type` (top 10)
- Also COUNT(*) over time (interval 3600000ms) for trend
- If non-zero: `search_datadog_rum_events` to inspect samples; cluster by `@error.message`

### Phase 4 — RUM Performance

For each mobile/web app in `config.apps`:

- `aggregate_rum_events` with `query: @type:view service:<datadog_service> env:<env>`
- Compute: P95 of `@view.loading_time`, COUNT(*), group by `@view.name` (top 15, sorted by P95 desc)
- Flag P95 > 3000ms as warning, > 6000ms as critical
- Check action error rates: `@type:action @action.error.count:>0`, group by `@action.name`

### Phase 5 — Log Errors

For each service in `config.apps` + `config.backend_services`:

- `search_datadog_logs` with `use_log_patterns: true`, `query: service:<name> env:<env> status:(error OR critical OR emergency)`
- Cluster on message to surface recurring errors
- Also `analyze_datadog_logs`: `SELECT status, count(*) FROM logs GROUP BY status ORDER BY count(*) DESC`

### Phase 6 — APM Span Health

For each service in `config.apps` (backend only) + `config.backend_services`:

- `aggregate_spans` with `query: service:<name> env:<env>`
- Compute: COUNT(*) and sum of errors, group by `resource_name` (top 20)
- Also P95 `duration` by resource. Flag P95 > 2s as warning, > 5s as critical.
- If error spans exist: `search_datadog_spans` with `status:error`, inspect samples.

### Phase 7 — Downstream Dependencies

For each service in `config.apps` + `config.backend_services`:

- `search_datadog_service_dependencies` with `direction: downstream`
- For each downstream service: `aggregate_spans` with COUNT, P95, error count
- Flag services with error rate > 1% or P95 > 3s

---

## Output

Produce a structured summary with a `<json-render>` status overview followed by a plain-text details section.

### Status dashboard (json-render)

- Heading: "App Health Check — <config.window>"
- A `StatusLine` per phase: `success` (clean), `warning` (minor), `error` (critical)
- Metrics: total crashes, impacted users, total RUM errors, P95 view loading time
- A `Callout` for any critical finding

### Details section (plain text)

- Per-phase findings with concrete numbers, service names, error messages
- Crashlytics: one bullet per open issue with event count, impacted users, version range, signals, and Firebase console link (use `issue.uri`)
- Datadog: relevant incident/monitor/span links
- Recommended next steps for each issue

If everything is clean: "No issues found — all signals healthy across <N> apps and <M> backend services."

---

## Constraints

- Do not speculate about root causes without data. Ground every claim in a query result.
- Do not run non-readonly tools. This skill is pure observability — it reads Crashlytics, Datadog, and the config file.
- The only file it writes is `.factory/app-observability.json` during setup.
- Never push git changes. If the user asks to commit the config, stage and prepare a commit; they push on their own.
