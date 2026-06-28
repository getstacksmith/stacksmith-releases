# Stacksmith (for Codex)

This file teaches Codex to author `.stacksmith.yml` config files and to drive a running
Stacksmith macOS app through its MCP server. It is self-contained; everything Codex needs is
below.

Stacksmith runs a whole local development stack from one `.stacksmith.yml` file at the project
root. Every runnable thing has a role (**app**, **worker**, **job**, or **tunnel**), plus
optional **links**. Stacksmith starts components in dependency order, runs health checks,
tracks readiness and ports, streams per-component logs, and produces a deterministic failure
diagnosis.

There are two jobs. Work out which the user needs:

1. **Authoring** `.stacksmith.yml` (plain file editing; the app need not be running).
2. **Controlling and inspecting** a running stack via the MCP tools (needs the app open, a
   project selected, and the relevant MCP permission enabled).

## Authoring `.stacksmith.yml`

Stacksmith validates strictly and **rejects unknown fields**, so use only the keys below. Put
the file at the project root, named exactly `.stacksmith.yml`.

```yaml
version: 1                 # required, only 1 is supported
name: My Awesome App       # required

apps:                      # long-running foreground services
  api:
    name: API Server       # optional display name
    command: cargo run --bin api   # required
    cwd: .                 # working dir (defaults to project root)
    port: 8080             # apps only: the one TCP port this app owns
    url: http://localhost:${apps.api.port}
    health: ${apps.api.url}/health   # HTTP probe (passes on 2xx)
    required_env: [DATABASE_URL]     # env var NAMES that must be present
    startup_timeout: 30    # seconds
    shutdown_timeout: 10   # seconds
    depends_on: {}         # map of id: condition

workers:                   # long-running background processes
  worker:
    command: cargo run --bin worker
    health:                # command probe form (passes on exit 0)
      command: pg_isready -h localhost -p 5432   # required
      interval: 5          # seconds between probes
      timeout: 2           # seconds before a probe is killed
      start_period: 5      # grace seconds before failures count
      cwd: .               # probe working dir (defaults to component cwd)
    depends_on:
      api: healthy
      migrate: completed

jobs:                      # one-shot commands that run to completion
  migrate:
    command: cargo run --bin migrate
    auto_start: false      # true (default) runs during Start All; false = on demand only
    depends_on:
      api: port_listening

tunnels:                   # expose a public URL for a local app
  stripe_webhook:
    command: ngrok http --url=${tunnels.stripe_webhook.public_url} ${apps.api.port}
    public_url: https://example.ngrok-free.app   # required
    forwards_to: api       # required, must name an existing app
    depends_on:
      api: healthy

links:                     # named URLs, not runnable processes
  api: ${apps.api.url}
```

**Allowed fields per role.** Common to all: `command` (required), `cwd`, `name`, `depends_on`,
`required_env`, `startup_timeout`, `shutdown_timeout`. Apps add `port`, `url`, `health`.
Workers add `health`. Jobs add `auto_start`. Tunnels add `public_url`, `forwards_to`, `health`.

**Rules to respect.**

- Component ids (the YAML keys) are unique **across all roles**, not just within a group.
- At least one runnable component is required.
- Only apps own a `port`. Tunnels do not own the forwarded port.
- `forwards_to` must reference an existing app.

**Dependency conditions** (in `depends_on`, AND-combined when there is more than one):

- `started`: the dependency process launched. (any component)
- `healthy`: the dependency passed its health check. (the target must define `health`)
- `port_listening`: the dependency owns and listens on its port. (apps)
- `completed`: the dependency job exited successfully. (jobs only)

A component depending on a manual job (`auto_start: false`) with `completed` stays blocked
until that job is run and succeeds; Start All will not run it automatically.

**Interpolation.** `${...}` references another scalar value in the config and resolves before
validation. Useful refs: `${apps.<id>.port}`, `${apps.<id>.url}`, `${apps.<id>.health}`,
`${tunnels.<id>.public_url}`, `${links.<id>}`. No shell, no env expansion, no expressions.
Missing references and cycles are errors.

**After editing**, validate: if the project is loaded in a running app, call
`stacksmith_validate_loaded_project`. Otherwise tell the user to reload the project in
Stacksmith, which reports issues with line numbers.

## Controlling and inspecting via MCP

**Prerequisites:** the app is open, a project is selected, and the matching MCP permission is
enabled in Stacksmith → Settings → MCP (read-only inspection for read tools; component control,
a separate toggle that is off by default, for the control tools). If a prerequisite is missing
the tool returns an error; relay it with the fix rather than retrying.

**Read tools** (always safe, no permission beyond read-only):

- `stacksmith_get_project_status` (no args): project overview.
- `stacksmith_list_components` (no args): all components, ids, roles, state.
- `stacksmith_get_component_status` (`component_id`): full state for one component.
- `stacksmith_get_recent_logs` (`component_id`, optional `limit` 1-500, default 100): log tail.
- `stacksmith_get_diagnosis` (no args): affected component, evidence, recommended action.
- `stacksmith_validate_loaded_project` (no args): validity and current issues.

**Control tools** (need component-control permission; mutate local state):

- `stacksmith_start_component` (`component_id`, optional `wait_until_ready` default true,
  `timeout_seconds` 1-120 default 30): start one app/worker/tunnel.
- `stacksmith_stop_component` (`component_id`, optional `wait_until_stopped` default true,
  `timeout_seconds` 1-120 default 30): stop one app/worker/tunnel.
- `stacksmith_restart_component` (same args as start): restart one app/worker/tunnel; cannot
  restart jobs.
- `stacksmith_run_job` (`component_id`, optional `wait_for_completion` default true,
  `timeout_seconds` 1-900 default 120, `log_limit` 0-500 default 200): run or rerun a job. A
  timeout does not cancel the job.

**How to behave:**

- **Read before acting.** Call `stacksmith_list_components` first so you use real ids and
  current state. Never invent a `component_id`.
- **Each control tool acts on one component only.** It does not start dependencies or stop
  dependants. If a target is blocked on a dependency, start that dependency first.
- **Confirm before mutating.** `stop`, `restart`, and `run_job` can interrupt in-flight work
  or change databases, files, and services. Ask the user first unless they already said to go
  ahead.
- **Prefer the built-in diagnosis** over re-deriving failures from raw logs; pull logs only for
  extra detail.

## Setup

Register the bundled helper, then enable permissions in the app:

```bash
codex mcp add stacksmith -- \
  /Applications/Stacksmith.app/Contents/Helpers/stacksmith-mcp

codex mcp get stacksmith   # verify
```

Open Stacksmith → Settings → MCP, enable read-only inspection, then enable component control
separately if mutating tools are wanted. Select a project before calling project tools.

**Troubleshooting.** If every tool errors, the app is not open or no project is selected. If
only control tools error, the component-control permission is off. To check the bridge socket:

```bash
ls -l "$HOME/Library/Application Support/Stacksmith/runtime.sock"
```
