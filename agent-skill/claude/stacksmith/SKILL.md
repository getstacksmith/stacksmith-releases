---
name: stacksmith
description: >-
  Author, edit, and validate Stacksmith `.stacksmith.yml` config files and drive a
  running Stacksmith macOS app through its MCP server. Use this skill whenever the user
  mentions Stacksmith, a `.stacksmith.yml` file, or wants to define, run, start, stop,
  restart, or diagnose a local development stack made of apps, workers, jobs, and tunnels
  from one YAML file. Also use it when they ask to add a service, worker, one-shot job, or
  tunnel to their local stack, set up health checks or dependency-aware startup ordering,
  wire up port and URL interpolation, or inspect the status, logs, readiness, or failure
  diagnosis of local services that Stacksmith manages. Trigger even when the user does not
  say the word "skill" and even when they only describe the symptom (for example "my worker
  won't start until the API is healthy" or "start my local API and tell me when it's ready").
---

# Stacksmith

Stacksmith is a native macOS app that runs a whole local development stack from a single
`.stacksmith.yml` file in the project root. Every runnable thing has a role (an **app**,
**worker**, **job**, or **tunnel**), plus optional **links**. Stacksmith starts components
in dependency order, runs health checks, tracks readiness and ports, streams per-component
logs, and produces a deterministic failure diagnosis.

You help with two distinct jobs. Work out which one the user needs before acting:

1. **Authoring** the `.stacksmith.yml` file (create, edit, validate). This is plain file
   editing and does not need the app running.
2. **Controlling and inspecting** a running stack through the Stacksmith **MCP server**
   (status, logs, diagnosis, and start/stop/restart/run-job). This needs the app open with
   the project selected and the MCP server enabled.

The two jobs share vocabulary, so read the relevant reference before you act rather than
relying on memory, since the config rejects unknown fields and the MCP tools act on real
component IDs.

## Authoring `.stacksmith.yml`

**Read [references/yaml-schema.md](references/yaml-schema.md) before writing any config.**
It is the exact schema: the field set per role, health-check forms, dependency conditions,
interpolation rules, and a full annotated example. Stacksmith validates strictly and
**rejects unknown keys**, so guessing field names will produce a config that fails to load.

When creating or editing config:

- Put the file at the **project root**, named exactly `.stacksmith.yml`. Start with
  `version: 1` and a `name`.
- Give every component a unique `id` (the YAML key). IDs must be unique **across all roles**,
  not just within a group.
- Prefer interpolation (`${apps.api.port}`, `${apps.api.url}`) over repeating literal ports
  and URLs, so a single edit stays consistent everywhere.
- Add `depends_on` with an explicit condition (`started`, `healthy`, `port_listening`,
  `completed`) instead of leaving startup order to chance.
- After writing, **validate**. If the project is loaded in a running app and the MCP server
  is available, call `stacksmith_validate_loaded_project`. Otherwise tell the user to reload
  the project in Stacksmith, which surfaces any issues with line numbers.

## Controlling and inspecting via MCP

**Read [references/mcp-tools.md](references/mcp-tools.md)** for the full tool list, their
arguments, safety classification, setup, and troubleshooting.

Core principles, because the tools are deliberately narrow:

- **Read before you act.** Call `stacksmith_list_components` or `stacksmith_get_project_status`
  first so you use real component IDs and current state. Never invent an ID.
- **The control tools act on one component only.** `start_component`, `stop_component`,
  `restart_component`, and `run_job` do **not** start dependencies or stop dependants. If a
  target is blocked waiting on a dependency, start that dependency first.
- **Confirm before mutating.** `stop`, `restart`, and `run_job` can interrupt in-flight local
  work or change databases, files, and services. Ask the user before running them unless they
  have already told you to proceed. Read-only tools (status, list, logs, diagnosis, validate)
  are always safe.
- **Prefer the built-in diagnosis.** When something fails, call `stacksmith_get_diagnosis`
  rather than re-deriving the cause from raw logs. It returns the affected component, the
  evidence, and a recommended action. Pull `stacksmith_get_recent_logs` only when you need
  more detail than the diagnosis gives.
- **The app owns the runtime.** If the app is not open, no project is selected, or the
  required permission is off, the tool returns an error. Relay that to the user with the fix
  (open the app, select the project, or enable the permission in Settings) rather than retrying
  blindly.

## Setup notes

The MCP control tools require the **component-control permission**, which is separate from
read-only access and off by default. Both are enabled in **Stacksmith → Settings → MCP**.
Registration for Claude and Codex, the helper path, and the socket check live in
[references/mcp-tools.md](references/mcp-tools.md).
