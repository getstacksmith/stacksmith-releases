# `.stacksmith.yml` schema

The complete, exact schema Stacksmith accepts. Stacksmith validates strictly and **rejects
unknown fields**, so only use the keys listed here. The file lives at the project root and is
named exactly `.stacksmith.yml`.

## Contents

- [Top level](#top-level)
- [Component roles and fields](#component-roles-and-fields)
- [Health checks](#health-checks)
- [Dependencies](#dependencies)
- [Interpolation](#interpolation)
- [Lifecycle and readiness](#lifecycle-and-readiness)
- [Validation rules to respect](#validation-rules-to-respect)
- [Full annotated example](#full-annotated-example)

## Top level

```yaml
version: 1            # required. Only version 1 is supported today.
name: My Project      # required. Display name for the project.
apps:      { ... }    # optional group of app components
workers:   { ... }    # optional group of worker components
jobs:      { ... }    # optional group of job components
tunnels:   { ... }    # optional group of tunnel components
links:     { ... }    # optional map of named URLs
```

Allowed top-level keys: `version`, `name`, `apps`, `workers`, `jobs`, `tunnels`, `links`.
The config must declare **at least one** runnable component across `apps`, `workers`, `jobs`,
or `tunnels`.

Each group is a mapping of `component_id: { fields }`. The component id is the YAML key
(for example `api` below). Ids must be **globally unique across every role**.

## Component roles and fields

### Fields common to all roles

| Field | Required | Meaning |
|---|---|---|
| `command` | yes | Shell command that runs the component. |
| `cwd` | recommended | Working directory, relative to the project root or absolute. Defaults to the project root if omitted. |
| `name` | no | Display name shown in the UI. Defaults to the id. |
| `depends_on` | no | Map of `component_id: condition` (see [Dependencies](#dependencies)). |
| `required_env` | no | List of environment variable **names** that must be present. Stacksmith reports a startup issue for any that are missing. |
| `startup_timeout` | no | Seconds to allow for startup before treating it as failed. |
| `shutdown_timeout` | no | Seconds to allow for graceful shutdown before force-stopping. |

### `apps`

Long-running foreground services. Allowed fields: the common ones **plus** `port`, `url`,
`health`.

| Field | Meaning |
|---|---|
| `port` | The single TCP port this app owns. Enables port preflight, conflict detection, and release checks. **Only apps own ports.** |
| `url` | Project URL, usually built from the port with interpolation. |
| `health` | Readiness probe. See [Health checks](#health-checks). |

### `workers`

Long-running background processes and queue runners. Allowed fields: the common ones **plus**
`health`. Workers do not own a port.

### `jobs`

One-shot commands that run to completion. Allowed fields: the common ones **plus**
`auto_start`.

| Field | Meaning |
|---|---|
| `auto_start` | `true` (default) runs the job during Start All. Set `false` for migrations and other commands that should only run on an explicit Run action or via `stacksmith_run_job`. |

Jobs have no `health` and no `port`.

### `tunnels`

Auxiliary processes that expose a public URL for a local app. Allowed fields: the common ones
**plus** `public_url`, `forwards_to`, `health`.

| Field | Required | Meaning |
|---|---|---|
| `public_url` | yes | The public URL the tunnel exposes. |
| `forwards_to` | yes | The id of the **app** this tunnel forwards to. Must reference an existing app. |
| `health` | no | Optional readiness probe. |

A tunnel does **not** own the forwarded app's port. The public URL belongs to the tunnel; the
local port stays owned by the app.

### `links`

A mapping of name to URL for project URLs that are not runnable processes. Values are strings
and may use interpolation.

```yaml
links:
  api: ${apps.api.url}
  docs: http://localhost:3000/docs
```

## Health checks

A `health` value gates readiness and is what satisfies the `healthy` dependency condition. It
takes one of two forms.

**HTTP health** is a URL string. The check passes on any `2xx` response.

```yaml
apps:
  api:
    command: cargo run --bin api
    port: 8080
    health: http://localhost:8080/health
```

**Command health** is a mapping. The check passes when the command exits `0`; a nonzero exit,
launch failure, or timeout is unhealthy. Probes run serially and never overlap. Timing fields
are in seconds.

```yaml
workers:
  worker:
    command: cargo run --bin worker
    health:
      command: pg_isready -h localhost -p 5432   # required
      interval: 5        # seconds between probes
      timeout: 2         # seconds before a probe is killed
      start_period: 5    # grace seconds before failures count
      cwd: .             # working dir for the probe; defaults to the component cwd
```

Allowed command-health fields: `command` (required), `cwd`, `interval`, `timeout`,
`start_period`.

## Dependencies

`depends_on` maps a component id to the condition that must hold before the dependent starts.
Listing more than one combines them with AND, so the component starts only once **every**
listed dependency is satisfied. Cycles across any roles are invalid.

| Condition | Holds when | Valid for |
|---|---|---|
| `started` | The dependency process has launched. | Any component |
| `healthy` | The dependency passed its health check. | Components that define `health` |
| `port_listening` | The dependency owns and is listening on its port. | Components with a `port` (apps) |
| `completed` | The dependency job exited successfully. | Jobs only |

```yaml
workers:
  worker:
    command: cargo run --bin worker
    depends_on:
      api: healthy
      migrate: completed
```

A `healthy` dependency must point at a component that actually defines a `health:` check. If a
component depends on a **manual** job (`auto_start: false`) with `completed`, Start All will not
run that job automatically; the dependent stays blocked until the job is run and succeeds.

## Interpolation

String values may reference other scalar values in the config with `${...}`. References resolve
after YAML decoding and before validation, so the resolved ports, URLs, paths, and conditions
are still checked. A single string may contain more than one reference. Missing references,
empty references, references that resolve to a non-scalar, and reference cycles are all
validation errors. Stacksmith does **not** evaluate shell syntax, expand environment variables,
or support conditionals or expressions inside `${...}`.

Commonly used references:

| Reference | Resolves to |
|---|---|
| `${apps.<id>.port}` | The app's configured port. |
| `${apps.<id>.url}` | The app's configured URL. |
| `${apps.<id>.health}` | The app's health-check URL. |
| `${tunnels.<id>.public_url}` | A tunnel's public URL. |
| `${links.<id>}` | A declared link URL. |

```yaml
apps:
  api:
    port: 8080
    url: http://localhost:${apps.api.port}
    health: ${apps.api.url}/health
tunnels:
  stripe_webhook:
    command: ngrok http --url=${tunnels.stripe_webhook.public_url} ${apps.api.port}
    public_url: https://example.ngrok-free.app
    forwards_to: api
```

## Lifecycle and readiness

- Apps, workers, and tunnels are **long-running**. Readiness is determined by health check
  first, then owned port, then a startup settle. Any unexpected exit is a failure, **including
  exit code 0**.
- Jobs are **one-shot**. A successful job enters `completed`, is skipped by Stop All, and can
  be run again explicitly. Use `auto_start: false` for jobs that should only run on demand.
- Display order in the app is apps, workers, tunnels, then jobs.

## Validation rules to respect

- `version` must be `1`; `name` is required.
- At least one runnable component is required.
- Component ids are globally unique across all roles.
- Unknown fields anywhere are errors. Match the allowed field set per role exactly.
- `command` is required on every component.
- `port` is only valid on apps.
- `public_url` and `forwards_to` are required on tunnels, and `forwards_to` must name an app.
- A `healthy` dependency must target a component that defines `health`.
- `completed` is only valid when depending on a job.

## Full annotated example

```yaml
version: 1
name: My Awesome App

apps:
  api:
    name: API Server
    command: cargo run --bin api
    cwd: .
    port: 8080
    url: http://localhost:${apps.api.port}
    health: ${apps.api.url}/health
    required_env:
      - DATABASE_URL
    startup_timeout: 30

workers:
  worker:
    name: Background Worker
    command: cargo run --bin worker
    cwd: .
    depends_on:
      api: healthy
      migrate: completed

jobs:
  migrate:
    name: Migrate Databases
    command: cargo run --bin migrate
    cwd: .
    auto_start: false        # run on demand, not during Start All
    depends_on:
      api: port_listening

tunnels:
  stripe_webhook:
    name: Stripe Webhook Tunnel
    command: ngrok http --url=${tunnels.stripe_webhook.public_url} ${apps.api.port}
    cwd: .
    public_url: https://guiding-lucky-tick.ngrok-free.app
    forwards_to: api
    depends_on:
      api: healthy

links:
  api: ${apps.api.url}
```
