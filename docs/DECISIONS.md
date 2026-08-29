# Architecture Decisions

This file records decisions that materially constrain implementation. Status values are `Proposed`, `Accepted`, `Superseded`, or `Rejected`.

When a decision changes, preserve the old entry, mark it `Superseded`, and link the replacement.

---

## ADR-001 — Use one overlay repository

**Status:** Accepted  
**Date:** 2026-08-29

### Context

The integration touches both Herdr and GSD-Pi. Maintaining full long-lived forks of both would duplicate upstream history, increase merge work, and make it difficult to identify which changes are integration-specific.

Both projects already expose extensibility surfaces:

- Herdr provides plugins plus a public CLI/socket API.
- GSD-Pi provides community extensions and a shared event bus.

The only missing surface identified so far is a generic way for an extension to execute the existing bundled subagent through an external runtime.

### Decision

Maintain one repository, `penggin/gsd-herdr`, containing:

- a GSD community extension;
- a Herdr plugin;
- a worker runner;
- a shared protocol;
- installer/update/rollback tooling;
- compatibility tests;
- a minimal version-specific GSD-Pi patch queue.

Do not maintain permanent runtime forks as the default architecture.

### Consequences

Positive:

- integration code is centralized;
- upstream histories remain external;
- Herdr can usually update without patch ports;
- the GSD patch is visible and bounded;
- removing the patch later is straightforward.

Negative:

- installer/build tooling is more complex than consuming one forked binary;
- compatibility must be tested against two upstreams;
- patch delivery must be exact and reproducible.

---

## ADR-002 — Keep Herdr core unpatched initially

**Status:** Accepted  
**Date:** 2026-08-29

### Context

Required Herdr behavior includes workspace/tab/pane creation, command execution, output reading, key sending, agent/session/metadata reporting, release, snapshot, and plugin actions.

These capabilities appear available through Herdr's documented public plugin, CLI, and socket surfaces.

### Decision

Implement the integration using a Herdr plugin and public APIs. Keep `patches/herdr/` empty unless a reproduced requirement cannot be satisfied safely through those APIs.

### Consequences

- official Herdr binaries remain usable;
- upstream updates are capability-tested rather than patch-ported;
- a future Herdr patch requires a new explicit decision and evidence.

---

## ADR-003 — Add a generic GSD external-subagent backend seam

**Status:** Accepted, pending M0 packaging validation  
**Date:** 2026-08-29

### Context

GSD-Pi's bundled subagent implementation owns important behavior: parallelism, chains, retries, session forking, isolated worktrees, result parsing, usage accounting, run-store updates, and cancellation.

Replacing or copying the whole `subagent` tool in an external extension would require continuously mirroring those behaviors and relying on tool registration/load order.

### Decision

Patch GSD-Pi only to expose a generic external execution seam from the existing bundled subagent path. The seam uses the shared extension event bus and contains no Herdr-specific code.

The external backend receives the launch plan and structured output/cancellation callbacks. GSD continues to parse and own the result.

### Consequences

- the bundled subagent remains authoritative;
- patch scope can remain small;
- every supported subagent mode must be routed through the seam consistently;
- exact upstream-version patch ports and parity tests are required;
- the patch can be removed if an equivalent upstream hook appears.

### Rejected alternative

Register another tool named `subagent` from the community extension and rely on load order. Rejected because it duplicates upstream implementation and is fragile.

---

## ADR-004 — Keep subagents in JSON mode

**Status:** Accepted  
**Date:** 2026-08-29

### Context

GSD-Pi launches subagents in JSON mode so the parent can recover assistant messages, tool results, usage, model metadata, stop reasons, and errors.

Running an interactive TUI instead would make structured parent-side processing difficult and could alter orchestration semantics.

### Decision

Continue using the existing GSD launch plan and `--mode json`. The worker runner captures the complete JSONL stream and relays complete records to GSD's existing parser.

### Consequences

- semantic behavior remains aligned with local execution;
- worker panes need a separate filtered renderer;
- raw JSONL artifacts may contain sensitive project information and require protected storage.

---

## ADR-005 — Do not render token deltas or raw JSON

**Status:** Accepted  
**Date:** 2026-08-29

### Context

Mirroring JSON-mode stdout directly into a terminal produces large raw `message_update` records for small text deltas. It is noisy, expensive to render, and not useful for human monitoring.

### Decision

Worker panes display only:

- worker identity;
- lifecycle state;
- concise tool starts/completions;
- retry/blocked/failure information;
- elapsed time;
- bounded final summary where configured.

The runner does not display raw JSON or token-level `message_update` events.

### Consequences

- panes remain readable during parallel execution;
- complete evidence remains in `stdout.jsonl`;
- a full worker TUI is explicitly out of scope for the first stable release.

---

## ADR-006 — Use a dedicated Node worker runner

**Status:** Accepted  
**Date:** 2026-08-29

### Context

A shell pipeline such as `child | tee stdout.jsonl` duplicates raw JSON to the terminal and introduces quoting, pipeline-exit, process-group, and cross-platform complexity.

The integration already targets a Node-based GSD runtime.

### Decision

Launch a fixed `gsd-herdr-worker` process in each Herdr pane. It reads a validated launch artifact and spawns the GSD child with argv arrays and `shell: false`.

The runner owns stream capture, filtered rendering, Herdr worker state, heartbeat, signal escalation, and exit artifacts.

### Consequences

- no shell interpolation of task/model/path values;
- reliable process and stream control;
- another versioned component must be installed and tested;
- the runner becomes a security-sensitive boundary.

---

## ADR-007 — Separate main-pane authority from worker authority

**Status:** Accepted  
**Date:** 2026-08-29

### Context

GSD subagents inherit most parent environment variables. Without guards, a headless child may report its lifecycle against the parent's Herdr pane and incorrectly mark a still-working main session idle or replace its session identity.

### Decision

- The root GSD reporter activates only for `ctx.mode === "tui"` and when `GSD_SUBAGENT_CHILD !== "1"`.
- The worker runner reports worker-pane authority.
- The child GSD process does not independently claim either authority through the root reporter.
- Parent Herdr-managed variables are stripped and worker-pane values are reapplied before child launch.

### Consequences

- main and worker state cannot overwrite each other;
- separate sequence domains and release behavior are required;
- worker monitoring remains available even though the root extension ignores child processes.

---

## ADR-008 — Monitoring failure is fatal in the default production profile

**Status:** Accepted  
**Date:** 2026-08-29

### Context

A silent fallback from failed pane launch to local child execution allows a subagent to continue modifying a project without the required visible pane. It may also create duplicate execution if the external launch outcome is ambiguous.

### Decision

Default configuration:

```json
{
  "required": true,
  "fallback": "error"
}
```

Fallback to local execution is permitted only under explicit optional configuration and only before any external process may have started.

### Consequences

- availability failures are visible and actionable;
- monitored execution becomes part of correctness rather than decoration;
- users who prefer availability over observability may explicitly opt into local fallback.

---

## ADR-009 — Use a persistent worker-pane pool

**Status:** Proposed  
**Date:** 2026-08-29

### Context

Creating a new tab or arbitrary splits for every dispatch would cause unbounded UI growth. GSD currently limits active parallel subagent concurrency, making a bounded pane pool practical.

### Decision

Associate one worker tab/pool with one root GSD session. Default capacity is four panes. Parallel tasks use available slots; chain steps and retries reuse stable panes where possible.

### Validation required

M4 must verify:

- usability of one-/two-/four-pane layouts;
- correct behavior when more tasks are queued than visible slots;
- retention/reuse policy;
- detach/reattach persistence;
- multi-project/session separation.

### Consequences

- predictable UI footprint;
- more complex slot state and reconciliation;
- completed pane reset must safely clear prior metadata/authority.

---

## ADR-010 — Use durable versioned artifacts

**Status:** Accepted  
**Date:** 2026-08-29

### Context

Temporary files alone are insufficient for long-running sessions, parent crashes, detach/reattach, and Herdr restart reconciliation.

### Decision

Store launch, stdout, stderr, state, heartbeat, and exit evidence under a versioned integration-owned state root. Publish mutable state atomically; treat the exit artifact as immutable final evidence.

### Consequences

- crash/orphan diagnosis is possible;
- storage retention and cleanup become required features;
- artifacts need strict permissions and redaction-aware support tooling.

---

## ADR-011 — Install side-by-side and activate atomically

**Status:** Accepted  
**Date:** 2026-08-29

### Context

Patching an active upstream installation in place makes rollback difficult and risks leaving a half-updated runtime.

### Decision

Stage and test each stack version in a separate directory. Activate by changing version pointers and integration-owned links. Preserve the prior known-good version for rollback.

### Consequences

- updates and rollback are reproducible;
- additional disk space is required;
- installer ownership and transaction locking must be implemented carefully.

---

## ADR-012 — Capability-check Herdr and fingerprint GSD-Pi

**Status:** Accepted  
**Date:** 2026-08-29

### Context

Herdr exposes a public API schema, so behavior can be checked directly. GSD-Pi requires a source patch whose safe application depends on exact implementation structure.

### Decision

- Validate Herdr using required API capabilities plus tested version metadata.
- Validate GSD-Pi using exact version/commit, file fingerprints, patch checks, and behavioral tests.

### Consequences

- Herdr can often update without code changes;
- each supported GSD-Pi version needs a specific patch manifest;
- clean patch application is necessary but not sufficient for promotion.

---

## ADR-013 — Target macOS arm64 first

**Status:** Accepted  
**Date:** 2026-08-29

### Context

The initial user environment is macOS arm64. Process-group behavior, filesystem paths, and E2E Herdr execution can be validated deeply on one platform before broadening support.

### Decision

The first stable release supports macOS arm64. Code should avoid unnecessary platform lock-in, but Windows and Linux support are not release requirements until explicitly planned.

### Consequences

- smaller initial test matrix;
- process signaling may use Unix process-group semantics;
- platform assumptions must still be isolated and documented for later ports.

---

## ADR-014 — Do not choose a project license until implementation boundaries are finalized

**Status:** Proposed  
**Date:** 2026-08-29

### Context

The repository will contain original integration code and may contain patches derived from MIT-licensed GSD-Pi source. Herdr is Apache-2.0 licensed, but the initial architecture intends to call public APIs rather than copy core code.

### Decision

Delay selecting the repository's own license until M0 confirms what upstream-derived material, if any, will be committed. Preserve all source attribution and license obligations meanwhile.

### Validation required

Before distributing implementation artifacts:

- inventory copied/modified upstream code;
- choose a compatible project license;
- add `LICENSE` and any required notices;
- mark modified Apache-licensed files if any are distributed.
