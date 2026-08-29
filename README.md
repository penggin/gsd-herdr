# gsd-herdr

Persistent, observable GSD-Pi subagents running in Herdr-managed panes.

> **Project status:** documentation and technical-design phase. No production installation is available yet.

## Why this exists

GSD-Pi already has rich subagent orchestration: parallel and chained tasks, retries, context forking, isolated worktrees, result parsing, usage accounting, and cancellation. Herdr provides persistent terminal workspaces, panes, agent-aware state, detach/reattach, and a programmable CLI/socket API.

`gsd-herdr` connects those capabilities without maintaining long-lived full forks of either upstream project.

The intended experience is:

```text
Workspace: project

Tab: GSD
┌────────────────────────────────────────────┐
│ Main GSD TUI                               │
│ M04 / S02 / T03 · executing               │
│ Workers: 3 running, 1 queued               │
└────────────────────────────────────────────┘

Tab: GSD Workers
┌──────────────────────┬──────────────────────┐
│ falcon / scout       │ cedar / researcher   │
│ WORKING              │ WORKING              │
│ → bash git diff      │ → read STATE.md      │
├──────────────────────┼──────────────────────┤
│ harbor / reviewer    │ spruce / security    │
│ RETRYING 2/3         │ QUEUED               │
└──────────────────────┴──────────────────────┘
```

Raw Pi JSONL remains available to GSD for structured result processing, but worker panes show only concise, human-readable lifecycle and tool activity.

## Architecture in one paragraph

The repository ships a GSD community extension, a Herdr plugin, a worker runner, a shared protocol, and installation/compatibility tooling. A very small, Herdr-agnostic GSD-Pi patch exposes an external subagent execution seam. The extension answers that seam, allocates Herdr panes, and launches the worker runner. The runner starts the existing GSD child in JSON mode, stores complete JSONL and stderr artifacts, sends structured records back to GSD, and renders filtered status in the pane. Herdr itself remains unpatched unless a proven core limitation is discovered.

## Design goals

- Preserve GSD-Pi's existing orchestration and result semantics.
- Give every active subagent an observable Herdr pane.
- Keep the main GSD pane and worker panes accurately classified.
- Avoid raw token-delta JSON flooding worker terminals.
- Survive Herdr detach/reattach and diagnose crash/orphan conditions.
- Fail visibly when monitoring is required but unavailable.
- Adopt upstream Herdr and GSD-Pi updates with minimal patch maintenance.
- Support reproducible installation, canary testing, and rollback.

## Non-goals for the first stable release

- Reimplementing GSD-Pi's bundled `subagent` tool.
- Rendering a full independent GSD/Pi TUI in each worker pane.
- Token-by-token worker output.
- Cross-host distributed execution.
- Windows support.
- Permanent full forks of Herdr or GSD-Pi.

## Initial target

| Component | Baseline |
|---|---|
| Platform | macOS arm64 |
| Node.js | `>=22.18.0` |
| Herdr | `v0.8.2` |
| GSD-Pi | `v1.16.2` |
| Bridge protocol | `1` |

These versions are the first compatibility target, not a permanent support boundary.

## Repository strategy

```text
official Herdr
  + gsd-herdr Herdr plugin

official GSD-Pi
  + gsd-herdr GSD extension
  + minimal generic external-backend patch

gsd-herdr
  + worker runner
  + shared protocol
  + installer / doctor / update / rollback
  + compatibility and E2E tests
```

The long-term target is to remove the GSD-Pi patch after an equivalent generic extension seam exists upstream.

## Documentation

- [`PLANNING.md`](PLANNING.md) — canonical living plan, milestone status, risks, and progress log.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — component ownership, data flow, state machines, and recovery model.
- [`docs/INTEGRATION_CONTRACT.md`](docs/INTEGRATION_CONTRACT.md) — backend, runner, state, and artifact protocols.
- [`docs/CONFIGURATION.md`](docs/CONFIGURATION.md) — proposed user configuration and precedence rules.
- [`docs/OPERATIONS.md`](docs/OPERATIONS.md) — installation, runtime operation, updates, canary, rollback, and uninstall.
- [`docs/SECURITY.md`](docs/SECURITY.md) — trust boundaries, secrets, permissions, redaction, and process safety.
- [`docs/TESTING.md`](docs/TESTING.md) — test layers, parity requirements, failure injection, and CI matrices.
- [`docs/UPSTREAM_MAINTENANCE.md`](docs/UPSTREAM_MAINTENANCE.md) — patch queue and upstream update workflow.
- [`docs/DECISIONS.md`](docs/DECISIONS.md) — architectural decisions and their rationale.

## Current phase

The project is in **M0: repository foundation and feasibility validation**.

Current priorities:

1. Finish the initial design documentation.
2. Verify the released GSD-Pi runtime loading path and patch-delivery options.
3. Verify the required Herdr methods using its installed API schema.
4. Record the technical-spike results in `PLANNING.md`.
5. Bootstrap implementation packages only after those decisions are evidence-backed.

## Development rule

Every implementation session must begin by reading `PLANNING.md` and end by updating its task state, risks, decisions, progress log, and exact next action.

## Upstreams

- [open-gsd/gsd-pi](https://github.com/open-gsd/gsd-pi)
- [herdrdev/herdr](https://github.com/herdrdev/herdr)

## License

A project license will be selected before implementation artifacts are distributed. Third-party upstream code and patches must retain all license and attribution requirements of their source projects.
