# GSD–Herdr Living Plan

> **Status:** Documentation complete; technical feasibility validation next  
> **Last updated:** 2026-08-29  
> **Current milestone:** M0 — Repository foundation and feasibility validation  
> **Canonical rule:** Every implementation session starts by reading this file and ends by updating it.

## 1. Purpose

`gsd-herdr` is a single overlay/integration repository that connects [GSD-Pi](https://github.com/open-gsd/gsd-pi) to [Herdr](https://github.com/herdrdev/herdr) without maintaining permanent full forks of either project.

The integration must let GSD-Pi run its subagents in persistent Herdr-managed panes while preserving GSD-Pi's existing orchestration, retry, context, isolation, result parsing, and usage-accounting behavior.

The user-facing result should be:

- the main GSD session is visible in Herdr as `working`, `blocked`, or `idle`;
- each active subagent has a dedicated or pooled Herdr pane;
- each worker pane shows concise human-readable activity instead of raw Pi JSONL;
- detach/reattach does not stop the main GSD process or worker processes;
- failures, retries, cancellation, pane loss, and orphaned workers are explicit;
- upstream GSD-Pi and Herdr updates can be adopted without carrying large divergent forks.

## 2. Scope

### In scope

- A GSD-Pi community extension for main-session reporting and backend registration.
- A generic, Herdr-agnostic external-subagent-backend seam maintained as a minimal GSD-Pi patch until upstream provides one.
- A Herdr plugin for status, cleanup, worker navigation, and startup reconciliation.
- A worker runner that launches GSD subagents in JSON mode, captures structured output, renders filtered activity, and reports lifecycle state.
- A shared versioned protocol between the patch, extension, runner, and plugin.
- A version-aware installer, doctor, updater, canary workflow, rollback, and uninstaller.
- Compatibility testing against pinned stable versions and upstream heads.
- Durable local state and crash/orphan reconciliation.

### Out of scope for the first stable release

- Replacing GSD-Pi's orchestrator or bundled subagent implementation.
- Rendering a complete second GSD/Pi TUI inside every worker pane.
- Token-by-token output rendering.
- Modifying Herdr core unless a proven public API limitation requires it.
- Automatically adopting completed orphan results into a newly started parent session.
- Cross-host distributed execution.
- Windows support in the first release.
- Supporting arbitrary terminal multiplexers through this repository.

## 3. Design principles

1. **One integration repository, two upstreams.** Herdr and GSD-Pi remain external dependencies.
2. **Plugin first, patch last.** Use official public extension/plugin/API surfaces wherever possible.
3. **Minimal GSD patch.** The patch exposes a generic external execution seam and contains no Herdr-specific code.
4. **No silent fallback in production.** When monitoring is required, backend failure must fail the dispatch visibly instead of spawning an invisible local child.
5. **GSD remains authoritative.** GSD owns dispatch semantics, retries, result parsing, usage, session/fork logic, isolation, and merge decisions.
6. **Herdr owns terminals.** Herdr owns pane layout, persistence, focus, terminal input, and visible agent state.
7. **The runner owns translation.** Raw JSONL is preserved for GSD; only filtered lifecycle/activity information reaches the terminal.
8. **Version every boundary.** Protocols, artifacts, configuration, and compatibility declarations are explicitly versioned.
9. **Recover conservatively.** Never guess that a worker succeeded; retain evidence and mark uncertain states as orphaned or failed.
10. **Upstream-friendly implementation.** Changes should be separable into small, reviewable components that can later be proposed upstream.

## 4. Target architecture

```text
official GSD-Pi + minimal generic seam patch
        │
        │ external backend request (protocol v1)
        ▼
GSD community extension
        │
        ├── reports main GSD state to Herdr
        ├── manages the worker pane pool
        └── launches worker-runner instances
                │
                ▼
        Herdr-managed worker pane
                │
                └── gsd-herdr-worker
                        ├── launches GSD child with --mode json
                        ├── stores stdout.jsonl and stderr.log
                        ├── sends JSONL records back to the parent path
                        ├── renders filtered tool/lifecycle activity
                        └── writes state, heartbeat, and exit artifacts

official Herdr + gsd-herdr plugin
        ├── status/dashboard actions
        ├── focus-workers action
        ├── cleanup action
        └── startup reconciliation
```

Detailed component boundaries live in [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## 5. Initial compatibility baseline

The first implementation target is intentionally pinned:

| Component | Initial target |
|---|---|
| Platform | macOS arm64 |
| Node.js | `>=22.18.0` |
| Herdr | `v0.8.2` |
| GSD-Pi | `v1.16.2` |
| Bridge protocol | `1` |
| Configuration schema | `1` |
| Artifact schema | `1` |

These are development baselines, not an indefinite support promise. Before code is written, M0 must verify the exact released layouts and determine whether GSD-Pi requires a full source build or can use a safe resource overlay.

## 6. Repository layout target

```text
gsd-herdr/
├── PLANNING.md
├── README.md
├── docs/
├── packages/
│   ├── protocol/
│   ├── herdr-client/
│   ├── gsd-extension/
│   ├── worker-runner/
│   ├── herdr-plugin/
│   └── cli/
├── patches/
│   ├── gsd-pi/
│   └── herdr/
├── compat/
├── test/
└── .github/workflows/
```

The repository begins documentation-first. Package and patch directories are created only when their corresponding milestone starts.

## 7. Milestones

### M0 — Repository foundation and feasibility validation

**Goal:** Establish the documentation, decision record, compatibility assumptions, and the exact patch/build strategy.

**Status:** `IN PROGRESS`

Tasks:

- [x] M0.1 Create the repository.
- [x] M0.2 Add this living plan.
- [x] M0.3 Add project overview and documentation index.
- [x] M0.4 Document architecture and responsibility boundaries.
- [x] M0.5 Document protocol, configuration, security, testing, and upstream-maintenance plans.
- [ ] M0.6 Inspect the released GSD-Pi package and determine whether `src/resources` can be safely overlaid at runtime.
- [ ] M0.7 Verify the required Herdr API methods against the installed/released API schema.
- [ ] M0.8 Decide the patched-GSD distribution mechanism: resource overlay, source build, or both.
- [ ] M0.9 Create an executable technical-spike report with findings and revised estimates.

Exit criteria:

- Documentation describes one coherent architecture with no unresolved ownership overlap.
- Required Herdr methods are identified and capability-checkable.
- The exact GSD-Pi patch surface and delivery method are known.
- The next milestone can begin without revisiting repository topology.

### M1 — Main GSD session integration

**Goal:** Report the root GSD TUI session to Herdr accurately without tracking headless child sessions as the parent.

**Status:** `NOT STARTED`

Tasks:

- [ ] M1.1 Bootstrap the workspace and shared protocol package.
- [ ] M1.2 Implement a resilient Herdr client with bounded retries.
- [ ] M1.3 Create the GSD community extension and manifest.
- [ ] M1.4 Guard main reporting with `ctx.mode === "tui"`.
- [ ] M1.5 Exclude `GSD_SUBAGENT_CHILD=1` from root-session authority.
- [ ] M1.6 Report session identity, `working`, `blocked`, and `idle` states.
- [ ] M1.7 Report milestone/slice/task context as concise state messages.
- [ ] M1.8 Release pane authority on shutdown.
- [ ] M1.9 Add `/herdr-status` and `/herdr-doctor` commands.
- [ ] M1.10 Add unit and integration tests.

Exit criteria:

- The main GSD pane is correctly classified throughout a normal turn.
- A JSON-mode child cannot overwrite the parent pane's state or session identity.
- Extension reload and shutdown do not leave stale authority.

### M2 — Generic GSD external-subagent backend seam

**Goal:** Let an extension execute a bundled GSD subagent externally without copying or replacing the bundled subagent tool.

**Status:** `NOT STARTED`

Tasks:

- [ ] M2.1 Define backend resolve/request/result protocol v1.
- [ ] M2.2 Add a Herdr-agnostic resolver to GSD-Pi's subagent extension.
- [ ] M2.3 Preserve existing local and cmux behavior when no backend responds.
- [ ] M2.4 Support single, parallel, chain, background, retry, fork, and isolation paths.
- [ ] M2.5 Propagate cancellation and structured stdout/stderr callbacks.
- [ ] M2.6 Add strict `fallback: error` behavior.
- [ ] M2.7 Add fake-backend regression tests.
- [ ] M2.8 Produce version-specific patch manifests and fingerprints.

Exit criteria:

- A fake external backend produces the same GSD result shape as local execution.
- No-backend behavior is unchanged.
- The patch contains no Herdr import or product-specific logic.

### M3 — Worker runner

**Goal:** Run one GSD subagent in a Herdr pane while preserving raw JSONL and showing only readable activity.

**Status:** `NOT STARTED`

Tasks:

- [ ] M3.1 Define launch, state, heartbeat, and exit artifacts.
- [ ] M3.2 Spawn the child using argv arrays with `shell: false`.
- [ ] M3.3 Capture raw stdout JSONL and stderr without terminal duplication.
- [ ] M3.4 Parse chunked JSONL safely.
- [ ] M3.5 Render lifecycle, tool starts, retries, and final status.
- [ ] M3.6 Suppress token deltas and large payloads.
- [ ] M3.7 Redact secrets and limit preview lengths.
- [ ] M3.8 Report worker state to its own Herdr pane.
- [ ] M3.9 Implement heartbeat and atomic exit artifacts.
- [ ] M3.10 Implement SIGINT → SIGTERM → SIGKILL escalation.

Exit criteria:

- Raw JSON is present in the artifact but absent from pane output.
- Parent parsing receives complete JSONL records.
- Cancellation terminates the correct process group and writes a final artifact.

### M4 — Herdr backend and persistent pane pool

**Goal:** Execute all GSD subagent modes through a persistent Herdr worker tab and pane pool.

**Status:** `NOT STARTED`

Tasks:

- [ ] M4.1 Implement Herdr capability discovery.
- [ ] M4.2 Create/reuse one worker tab per main GSD session.
- [ ] M4.3 Create deterministic one-, two-, and four-pane layouts.
- [ ] M4.4 Implement pane-slot reservation, queueing, reuse, and retention.
- [ ] M4.5 Launch the worker runner through Herdr.
- [ ] M4.6 Relay worker JSONL to the GSD backend callbacks.
- [ ] M4.7 Keep retries and chain steps in stable panes.
- [ ] M4.8 Preserve main-pane focus by default.
- [ ] M4.9 Fail visibly if a required backend becomes unavailable.
- [ ] M4.10 Validate result parity with local execution.

Exit criteria:

- Single, parallel, and chain dispatches are visible and correctly completed.
- The parent receives the same final output, usage, error, and merge semantics.
- No unmonitored local fallback occurs in required mode.

### M5 — Herdr operations plugin

**Goal:** Make worker sessions discoverable and manageable without inspecting state files manually.

**Status:** `NOT STARTED`

Tasks:

- [ ] M5.1 Add the Herdr plugin manifest.
- [ ] M5.2 Add worker status and dashboard actions.
- [ ] M5.3 Add focus-workers and focus-failed-worker actions.
- [ ] M5.4 Add cleanup controls.
- [ ] M5.5 Add startup reconciliation.
- [ ] M5.6 Release stale agent authority safely.
- [ ] M5.7 Show orphaned or missing-pane workers explicitly.

Exit criteria:

- A user can locate every active, blocked, failed, and retained worker from Herdr.
- Stale state is diagnosed rather than silently discarded.

### M6 — Durability and recovery

**Goal:** Survive detach/reattach, pane closure, parent crashes, and Herdr restarts without invisible work.

**Status:** `NOT STARTED`

Tasks:

- [ ] M6.1 Add durable run/worker state under a versioned state root.
- [ ] M6.2 Add parent and worker heartbeats.
- [ ] M6.3 Reconcile state against Herdr `session.snapshot`.
- [ ] M6.4 Detect closed panes and lost processes.
- [ ] M6.5 Mark orphaned workers and retain evidence.
- [ ] M6.6 Prevent duplicate worker launch after reload/reconnect.
- [ ] M6.7 Add crash-injection and detach/reattach E2E tests.
- [ ] M6.8 Define bounded retention and cleanup policies.

Exit criteria:

- Every live worker is represented by a live pane and durable record.
- Every lost or uncertain worker has an explicit terminal state.
- Restart and reconnection do not create duplicate workers.

### M7 — Installation, updates, rollback, and release

**Goal:** Ship the stack as a reproducible, versioned installation with safe updates.

**Status:** `NOT STARTED`

Tasks:

- [ ] M7.1 Implement versioned side-by-side installation.
- [ ] M7.2 Install/link the GSD extension and Herdr plugin.
- [ ] M7.3 Build/apply the exact GSD patch set.
- [ ] M7.4 Add `install`, `doctor`, `status`, `update`, `switch`, `rollback`, and `uninstall` commands.
- [ ] M7.5 Add stable and canary channels.
- [ ] M7.6 Add upstream compatibility workflows.
- [ ] M7.7 Add macOS arm64 release packaging and checksums.
- [ ] M7.8 Publish installation and recovery documentation.

Exit criteria:

- A clean machine can install and validate the stack with documented commands.
- The previous known-good stack can be restored without rebuilding.
- Upstream incompatibility is detected before the active installation changes.

## 8. Current execution queue

The next tasks must be performed in order:

1. Inspect the released GSD-Pi package/runtime loading path (M0.6).
2. Verify the released Herdr API schema and required methods (M0.7).
3. Decide the patched-GSD distribution mechanism from evidence (M0.8).
4. Write the technical-spike report and revise estimates (M0.9).
5. Bootstrap code only after M0's patch/build decision is made.

## 9. Key technical decisions

| ID | Decision | Status |
|---|---|---|
| D001 | Use one overlay repository instead of permanent full forks | Accepted |
| D002 | Keep Herdr core unpatched initially; use its plugin and socket APIs | Accepted |
| D003 | Maintain only a generic GSD external-backend seam as a patch | Accepted, pending spike validation |
| D004 | Keep GSD child execution in JSON mode | Accepted |
| D005 | Do not display `message_update`/token deltas in worker panes | Accepted |
| D006 | Use a Node worker runner and argv-based spawn, not a shell pipeline | Accepted |
| D007 | In production, backend failure is an error rather than invisible local fallback | Accepted |
| D008 | Use a persistent worker-pane pool with a default maximum of four panes | Proposed; validate in M4 |
| D009 | Keep failed panes until manual cleanup; retain successful panes briefly | Proposed; validate with usage |
| D010 | Initial release targets macOS arm64 only | Accepted |

Architecture decisions are expanded in [`docs/DECISIONS.md`](docs/DECISIONS.md).

## 10. Risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| GSD-Pi subagent internals change frequently | Patch drift and semantic regressions | Tiny generic seam, exact fingerprints, upstream compatibility CI, result-parity tests |
| Herdr API evolves | Runtime failures | Capability-based checks via the installed schema, not version checks alone |
| Child inherits parent Herdr pane variables | Worker reports against the main pane | Strip Herdr-managed variables and reapply the worker pane context |
| Raw JSON floods the terminal | Unusable monitoring and high output cost | Capture JSONL to file; render only deduplicated lifecycle/tool activity |
| Parent crashes while workers continue | Orphaned code changes | Heartbeats, durable state, explicit orphan status, no automatic success assumptions |
| Pane closes without child exit artifact | Parent waits indefinitely | Pane/process monitoring and bounded failure transition |
| Secrets enter artifacts or metadata | Credential exposure | `0600` files, `0700` directories, redaction, no full environment in logs/metadata |
| Automatic patch applies incorrectly | Broken subagent execution | Exact version/fingerprint checks, `git apply --check`, no production fuzzy apply |
| Extension and patch protocol mismatch | Dispatch failure | Protocol version negotiation and fail-fast diagnostics |
| Upstream later adds a native backend hook | Duplicate mechanisms | Remove the patch while retaining extension/runner/plugin implementations |

## 11. Documentation map

- [`README.md`](README.md) — project overview and status.
- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — component boundaries and runtime flows.
- [`docs/INTEGRATION_CONTRACT.md`](docs/INTEGRATION_CONTRACT.md) — external backend, runner, and artifact contracts.
- [`docs/CONFIGURATION.md`](docs/CONFIGURATION.md) — configuration schema and defaults.
- [`docs/OPERATIONS.md`](docs/OPERATIONS.md) — installation, updates, rollback, and runtime operations.
- [`docs/SECURITY.md`](docs/SECURITY.md) — threat model and security requirements.
- [`docs/TESTING.md`](docs/TESTING.md) — unit, integration, compatibility, and E2E strategy.
- [`docs/UPSTREAM_MAINTENANCE.md`](docs/UPSTREAM_MAINTENANCE.md) — patch queue and upstream tracking.
- [`docs/DECISIONS.md`](docs/DECISIONS.md) — architectural decision records.

## 12. Plan update protocol

Every future development session must follow this sequence:

1. Read `PLANNING.md` before modifying code or documentation.
2. Confirm the current milestone, task IDs, dependencies, and exit criteria.
3. Inspect the latest upstream versions when the work depends on their current structure.
4. Make the smallest coherent change for one or more listed tasks.
5. Run the tests required by the affected component.
6. Update this file before ending the session:
   - check completed tasks;
   - update the current milestone;
   - add newly discovered tasks or risks;
   - record changed decisions;
   - add a dated progress-log entry;
   - identify the exact next task.
7. Do not mark a milestone complete until every exit criterion is supported by evidence.

## 13. Progress log

### 2026-08-29 — Project initialization

- Created the `penggin/gsd-herdr` repository.
- Chose a single overlay repository instead of maintaining full Herdr and GSD-Pi forks.
- Established a plugin-first architecture with one minimal GSD-Pi patch seam.
- Defined milestones M0–M7 and made `PLANNING.md` the canonical living plan.

### 2026-08-29 — Documentation foundation

- Added the project overview and documentation index in `README.md`.
- Defined component ownership, runtime flows, pane pooling, state authority, failure policy, and recovery in `docs/ARCHITECTURE.md`.
- Defined backend discovery/execution, runner, activity, artifact, and capability contracts in `docs/INTEGRATION_CONTRACT.md`.
- Defined the proposed schema-v1 configuration in `docs/CONFIGURATION.md`.
- Defined side-by-side installation, doctor, canary, update, rollback, cleanup, and uninstall behavior in `docs/OPERATIONS.md`.
- Defined the trust model, secret handling, path/process safety, redaction, and security tests in `docs/SECURITY.md`.
- Defined unit, contract, integration, compatibility, E2E, and canary test requirements in `docs/TESTING.md`.
- Defined exact GSD-Pi patch maintenance and capability-based Herdr tracking in `docs/UPSTREAM_MAINTENANCE.md`.
- Recorded ADR-001 through ADR-014 in `docs/DECISIONS.md`.
- Completed M0.3–M0.5.
- Next: inspect GSD-Pi's released runtime/package loading path for M0.6.
