# Stacksmith MCP server

The bundled `stacksmith-mcp` helper exposes the running app's state and controls over a local
Unix socket. The app remains the owner of all processes; the helper only forwards requests.

## Contents

- [Prerequisites](#prerequisites)
- [Tools](#tools)
- [Read tools](#read-tools)
- [Control tools](#control-tools)
- [Setup and registration](#setup-and-registration)
- [Troubleshooting](#troubleshooting)

## Prerequisites

For any tool to work:

1. The **Stacksmith app is open** (it hosts the local IPC server).
2. A **project is selected** in the app (project tools operate on the loaded project).
3. The matching **MCP permission is enabled** in Stacksmith → Settings → MCP:
   - **Read-only inspection** for the read tools.
   - **Component control** (separate toggle, off by default) for the control tools.

If a prerequisite is missing, the tool returns an error. Relay the error and its fix to the
user instead of retrying.

## Tools

| Tool | Class | Purpose |
|---|---|---|
| `stacksmith_get_project_status` | read | Concise overview of the selected project. |
| `stacksmith_list_components` | read | Every component in display order with role and state. |
| `stacksmith_get_component_status` | read | Lifecycle, readiness, dependency, port, exit, and failure state for one component. |
| `stacksmith_get_recent_logs` | read | Bounded chronological tail of stdout, stderr, and system logs for one component. |
| `stacksmith_get_diagnosis` | read | The deterministic diagnosis: affected component, evidence, recommended actions. |
| `stacksmith_validate_loaded_project` | read | Whether the loaded `.stacksmith.yml` is valid, with current issues. |
| `stacksmith_start_component` | control | Start one app, worker, or tunnel. |
| `stacksmith_stop_component` | control | Stop one app, worker, or tunnel. |
| `stacksmith_restart_component` | control | Restart one app, worker, or tunnel. |
| `stacksmith_run_job` | control | Run (or rerun) one one-shot job. |

**Always call a read tool first.** Use `stacksmith_list_components` to learn the real ids and
current state before any control call. Never invent a `component_id`.

The control tools are deliberately narrow: each one acts on **only the named component**. They
do not start dependencies, do not stop dependants, and do not touch jobs (except `run_job`). If
a component is blocked on a dependency, start that dependency first.

## Read tools

These never change runtime state and are always safe to call.

### `stacksmith_get_project_status`, `stacksmith_get_diagnosis`, `stacksmith_validate_loaded_project`, `stacksmith_list_components`

No arguments. Take an empty object `{}`.

### `stacksmith_get_component_status`

| Arg | Type | Required | Notes |
|---|---|---|---|
| `component_id` | string | yes | Id from `.stacksmith.yml`. |

### `stacksmith_get_recent_logs`

| Arg | Type | Required | Notes |
|---|---|---|---|
| `component_id` | string | yes | Id from `.stacksmith.yml`. |
| `limit` | integer | no | Entries to return. Default 100, range 1 to 500. |

## Control tools

These mutate local state and require the **component-control permission**. They can interrupt
in-flight local requests or work and can change databases, files, and services. **Confirm with
the user before calling `stop`, `restart`, or `run_job`** unless they have already told you to
proceed.

### `stacksmith_start_component`

Starts one app, worker, or tunnel. Does not start dependencies and does not restart something
already running.

| Arg | Type | Required | Notes |
|---|---|---|---|
| `component_id` | string | yes | App, worker, or tunnel id. |
| `wait_until_ready` | boolean | no | Wait for the existing readiness signal before returning. Default `true`. |
| `timeout_seconds` | integer | no | Readiness wait timeout, used only when waiting. Default 30, range 1 to 120. |

### `stacksmith_stop_component`

Stops one app, worker, or tunnel. Does not stop dependants and does not cancel jobs.

| Arg | Type | Required | Notes |
|---|---|---|---|
| `component_id` | string | yes | App, worker, or tunnel id. |
| `wait_until_stopped` | boolean | no | Wait for the stopped state before returning. Default `true`. |
| `timeout_seconds` | integer | no | Termination wait timeout. Default 30, range 1 to 120. |

### `stacksmith_restart_component`

Stops then starts one long-running component. Cannot restart one-shot jobs.

| Arg | Type | Required | Notes |
|---|---|---|---|
| `component_id` | string | yes | App, worker, or tunnel id. |
| `wait_until_ready` | boolean | no | Wait for readiness before returning. Default `true`. |
| `timeout_seconds` | integer | no | Readiness wait timeout. Default 30, range 1 to 120. |

### `stacksmith_run_job`

Runs (or reruns) one job declared in `.stacksmith.yml`. The command may modify local state,
databases, files, or services. Jobs are exclusive to this tool; `start_component` will not run
them.

| Arg | Type | Required | Notes |
|---|---|---|---|
| `component_id` | string | yes | Job id. |
| `wait_for_completion` | boolean | no | Wait for the job to complete or fail before returning. Default `true`. |
| `timeout_seconds` | integer | no | Completion wait timeout. Default 120, range 1 to 900. **A timeout does not cancel the job.** |
| `log_limit` | integer | no | Max recent log entries returned; 0 returns none. Default 200, range 0 to 500. |

## Setup and registration

The helper ships inside the installed app at a stable path:

```
/Applications/Stacksmith.app/Contents/Helpers/stacksmith-mcp
```

**Claude Code / generic MCP client** (`mcp.json`):

```json
{
  "mcpServers": {
    "stacksmith": {
      "command": "/Applications/Stacksmith.app/Contents/Helpers/stacksmith-mcp"
    }
  }
}
```

**Codex CLI:**

```bash
codex mcp add stacksmith -- \
  /Applications/Stacksmith.app/Contents/Helpers/stacksmith-mcp

codex mcp get stacksmith   # verify
```

After registering, open Stacksmith → Settings → MCP, enable read-only inspection, then enable
component control separately if mutating tools are wanted. Select a project before calling
project tools.

## Troubleshooting

- **All tools error / "not available":** the app is not open or the IPC server is not ready.
  Open Stacksmith and select a project.
- **Control tools error but read tools work:** the component-control permission is off. Enable
  it in Settings → MCP.
- **Verify the socket exists** when debugging the bridge:

```bash
ls -l "$HOME/Library/Application Support/Stacksmith/runtime.sock"
lsof -U | rg 'Stacksmith/runtime.sock'
```

The MCP settings page also has a non-mutating **Test Connection** action.
