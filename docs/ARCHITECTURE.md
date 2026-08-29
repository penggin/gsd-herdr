# Architecture

## 1. System boundary

`gsd-herdr` is an overlay stack between two independently maintained upstreams:

- **GSD-Pi** owns task orchestration and the semantic result of every subagent run.
- **Herdr** owns persistent terminals, workspaces, tabs, panes, input, focus, and visible agent state.
- **gsd-herdr** owns the integration protocol, pane allocation policy, worker process, observability translation, installation, and compatibility checks.

The integration deliberately avoids duplicating either upstream's responsibilities.

```text
┌──────────────────────────┐
│        GSD-Pi            │
│                          │
│ dispatch / retry / fork  │
│ isolation / result parse │
└────────────┬─────────────┘
             │ generic external backend seam
             ▼
┌──────────────────────────┐
│ gsd-herdr GSD extension  │
│                          │
│ main state reporter      │
│ Herdr backend provider   │
│ pane pool coordinator    │
└───────┬──────────┬───────┘
        │          │
        │          └────────────────────┐
        ▼                               ▼
┌───────────────────┐          ┌────────────────────┐
│ Herdr CLI/socket  │          │ durable run state  │
│ pane/tab/status   │          │ specs/logs/exit    │
└─────────┬─────────┘          └────────────────────┘
          │
          ▼
┌──────────────────────────┐
│ Herdr worker pane        │
│                          │
│ gsd-herdr-worker         │
│   └─ GSD child JSON mode │
└──────────────────────────┘
```

## 2. Components

### 2.1 Shared protocol

The protocol package defines versioned data exchanged by:

- the minimal GSD-Pi patch;
- the GSD extension;
- the worker runner;
- the Herdr plugin;
- the installer and doctor;
- test fixtures.

It contains no process or terminal implementation. Its job is to make compatibility failures explicit.

Primary contracts:

- backend discovery and selection;
- external execution requests and results;
- launch specifications;
- worker state and heartbeat records;
- exit artifacts;
- pane metadata;
- compatibility and capability declarations.

### 2.2 Minimal GSD-Pi patch

The patch adds a **generic external subagent execution seam** to the existing bundled subagent extension.

It must not:

- import Herdr code;
- know about workspaces, tabs, or panes;
- implement a second subagent tool;
- replace existing local or cmux behavior;
- decide how worker output is rendered.

It only:

1. asks the shared extension event bus whether an external backend is available;
2. passes the existing GSD launch plan and structured callbacks to the selected backend;
3. receives exit, stderr, abort, and external-runtime metadata;
4. feeds JSONL records into GSD's existing parser;
5. preserves current fallback behavior when no backend is registered;
6. supports a strict policy that forbids silent fallback after a selected backend fails.

The seam is intentionally general enough for eventual upstream adoption.

### 2.3 GSD community extension

The extension has two independent roles.

#### Root-session reporter

It reports the main GSD TUI process to Herdr:

- session identity;
- `working`, `blocked`, and `idle` state;
- active milestone, slice, and task context;
- shutdown/release.

It must only claim authority for the visible root TUI process. It therefore rejects:

- non-Herdr processes;
- JSON/RPC/print modes;
- processes marked `GSD_SUBAGENT_CHILD=1`.

#### External backend provider

It responds to the generic backend-discovery event and executes requests through Herdr.

Responsibilities:

- validate configuration;
- validate Herdr capabilities;
- create or reuse a worker tab and pane pool;
- create secure launch artifacts;
- launch the worker runner in the selected pane;
- relay JSONL output to GSD callbacks;
- relay stderr and final status;
- propagate cancellation;
- retain/release panes according to policy;
- fail explicitly when monitoring is required and unavailable.

### 2.4 Herdr client

The Herdr client is a small typed adapter over the public CLI and socket API.

It supports:

- capability discovery through the installed API schema;
- current workspace/tab/pane context;
- tab creation and lookup;
- pane split/layout/run/read/send-keys/rename/close;
- pane agent/session/metadata reporting;
- session snapshot and reconciliation;
- normalized errors and timeouts.

Most one-shot operations may use the CLI for portability and debuggability. Long-lived subscriptions or high-frequency state updates may use the socket API. The interface must hide that transport choice from callers.

### 2.5 Worker runner

The worker runner is the only process that directly owns a GSD subagent child inside a Herdr pane.

It receives a path to a launch specification and then:

1. validates the protocol and artifact paths;
2. reads one-time environment material;
3. replaces inherited main-pane Herdr variables with the worker pane's own context;
4. deletes the temporary environment artifact;
5. starts the GSD child with `shell: false` and argv arrays;
6. writes raw stdout to `stdout.jsonl`;
7. writes stderr to `stderr.log`;
8. parses complete JSONL records;
9. renders only deduplicated human-readable activity;
10. reports its state to the worker pane;
11. updates heartbeat and state artifacts;
12. forwards termination signals to the child process group;
13. writes an atomic exit artifact.

The runner does not interpret success beyond process and protocol evidence. GSD remains authoritative for semantic result parsing.

### 2.6 Herdr plugin

The Herdr plugin is an operations surface, not the execution core.

It provides:

- status and dashboard actions;
- focus-workers and focus-failed-worker actions;
- cleanup actions;
- startup reconciliation after Herdr restores a session;
- stale authority release;
- orphan and missing-pane diagnosis.

The plugin reads durable state and uses Herdr public APIs. It does not own GSD orchestration.

### 2.7 Stack CLI

The CLI manages the overlay stack:

- `install`;
- `doctor`;
- `status`;
- `update --check`;
- `update --canary`;
- `switch`;
- `rollback`;
- `uninstall`.

It performs exact compatibility checks and installs side-by-side versions. It never fuzzy-patches the active installation.

## 3. Runtime flows

### 3.1 Main-session startup

```text
User starts GSD inside Herdr
        │
        ▼
GSD loads gsd-herdr extension
        │
        ├─ verify HERDR_ENV and pane context
        ├─ reject child/headless process
        ├─ report agent session identity
        └─ report initial idle/working state
```

Main-state reporting is independent of subagent execution. It should work before the GSD-Pi backend patch exists.

### 3.2 External backend discovery

```text
Bundled subagent tool prepares launch plan
        │
        ▼
Patched resolver emits protocol-v1 request
        │
        ├─ zero providers → existing cmux/local path
        ├─ one provider  → use it
        └─ multiple      → deterministic priority selection
```

Backend discovery must be fast, deterministic, and side-effect free. Capability checks and pane creation occur during execution, not discovery.

### 3.3 Single worker execution

```text
GSD extension receives execution request
        │
        ├─ acquire pane slot
        ├─ write secure launch artifacts
        ├─ pane.run gsd-herdr-worker --spec ...
        └─ begin observing worker artifacts
                    │
                    ▼
            worker runner starts child
                    │
                    ├─ raw JSONL → stdout.jsonl
                    ├─ stderr → stderr.log
                    ├─ readable activity → pane
                    └─ lifecycle → Herdr state
                    │
                    ▼
            extension relays JSONL lines
                    │
                    ▼
            existing GSD event parser
                    │
                    ▼
            existing GSD result object
```

### 3.4 Parallel execution

The GSD orchestrator still controls concurrency. The backend maps active tasks onto a pane pool.

```text
GSD parallel tasks: 8
GSD active concurrency: 4
Herdr pane pool: 4

wave 1: tasks 1–4 occupy slots 1–4
wave 2: tasks 5–8 reuse eligible slots
```

The backend must not create additional concurrency beyond GSD's decision.

### 3.5 Chain execution

Chain steps should reuse one pane when practical so the user sees one continuous history.

```text
step 1 complete
──── chain step 2 ────
step 2 starts in same pane
```

GSD still substitutes `{previous}`, decides whether to continue, and constructs each launch plan.

### 3.6 Retry

Retries should normally reuse the same pane and preserve prior visible evidence.

```text
[00:28] failed · provider 503
──── retry 2/2 ────
[00:31] starting
```

A retry is a new child attempt but belongs to the same dispatch child record. The protocol therefore carries both stable child identity and attempt number.

### 3.7 Cancellation

```text
Parent abort signal
    │
    ├─ backend sends Ctrl-C to worker pane
    ├─ runner receives SIGINT
    ├─ runner forwards SIGINT to child process group
    ├─ grace timeout
    ├─ SIGTERM
    ├─ second grace timeout
    └─ SIGKILL if still alive
```

The runner writes an exit artifact even when escalation is required. The backend never assumes that sending Ctrl-C succeeded without observing termination evidence.

## 4. State authority

### 4.1 Main pane

Only the root GSD TUI extension instance may report main-pane agent authority.

```text
allowed:
  main GSD TUI in w1:p1

not allowed:
  JSON subagent child
  RPC mode
  print mode
  unrelated process inheriting pane variables
```

### 4.2 Worker pane

Only the worker runner occupying that pane may report worker authority. The child GSD process is marked `GSD_SUBAGENT_CHILD=1` and must not independently claim main or worker pane authority through the root extension.

### 4.3 Sequence ordering

All state reports include monotonically increasing sequence numbers scoped to one authority source. Late events must not overwrite newer state.

### 4.4 Release

Authority is released when:

- the main GSD session shuts down;
- a worker pane is reset for reuse;
- a worker is cleaned up;
- reconciliation determines that the authority source no longer exists.

## 5. Pane-pool model

### 5.1 Scope

One worker pool is associated with one root GSD session.

Stable identity includes:

- project/root working directory;
- root GSD session identity;
- Herdr workspace;
- worker tab identifier.

### 5.2 Default layout

The initial default is a dedicated worker tab with up to four panes.

```text
1 worker:  [A]

2 workers: [A | B]

3 workers: [A | B]
           [C |  ]

4 workers: [A | B]
           [C | D]
```

The exact layout may use `layout.apply` when available, with deterministic split fallback otherwise.

### 5.3 Slot lifecycle

```text
available
  → reserved
  → starting
  → running
  → retained-success | retained-failure | aborted | orphaned
  → resetting
  → available
```

A slot cannot be assigned to two attempts simultaneously. Reservation and transition writes must be atomic within the extension process.

### 5.4 Retention

Default proposal:

- success: retain 10 minutes, then eligible for reuse;
- aborted: retain 10 minutes;
- failure: retain until manual cleanup;
- blocked: never auto-close;
- orphaned: retain until resolved or manually cleaned.

## 6. Durable state

Default state root:

```text
~/.local/state/gsd-herdr/
├── installs/
├── sessions/
└── runs/
    └── <dispatch-id>/
        ├── run.json
        └── children/
            └── <child-id>/
                ├── launch.json
                ├── env.json          # one-time, deleted by runner
                ├── stdout.jsonl
                ├── stderr.log
                ├── state.json
                ├── heartbeat
                └── exit.json
```

Properties:

- directories are `0700`;
- sensitive files are `0600`;
- state and exit updates are write-temp + rename;
- IDs are generated, validated, and never taken directly from user text;
- cleanup resolves real paths and refuses to leave the configured state root.

## 7. Recovery model

### 7.1 Detach/reattach

No special recovery is required when the Herdr server and pane processes remain alive. The TUI client may disconnect and reconnect while work continues.

### 7.2 Main GSD process crash

Workers may remain alive because Herdr owns their panes.

Initial behavior:

- parent heartbeat becomes stale;
- worker pane reports `orphaned`;
- worker is not declared successful automatically;
- child may continue unless policy explicitly requests orphan termination;
- artifacts remain available for inspection;
- the Herdr plugin surfaces the condition.

Automatic result adoption into a new parent session is deferred beyond the first stable release.

### 7.3 Worker pane closed

If no exit artifact exists and the pane/process disappears, the backend resolves the attempt as failed with explicit `worker pane closed` evidence.

### 7.4 Herdr restart

The plugin startup hook reconciles durable state with `session.snapshot`:

- live pane + live process: restore association;
- record + missing pane: failed or orphaned;
- pane + missing record: leave untouched unless marked as gsd-herdr-owned;
- stale authority: release;
- expired retained success: cleanup.

### 7.5 Extension reload

Backend registration must be idempotent. Reload must not:

- create a second worker pool;
- duplicate state authority;
- duplicate active child launches;
- reset a running child to idle.

## 8. Failure policy

### Required mode

When `required: true` or `fallback: "error"`:

- missing Herdr capability fails dispatch;
- failed pane creation fails dispatch;
- failed worker launch fails dispatch;
- lost pane fails the attempt;
- no local invisible fallback is permitted.

### Optional mode

When explicitly configured for local fallback:

- backend discovery may decline;
- a pre-execution availability failure may return control to local execution;
- once an external child has started, failure must not create a duplicate local attempt.

## 9. Performance constraints

- Backend discovery should complete within one event-loop turn.
- State reports are deduplicated and rate limited.
- Human-readable pane output is capped; token deltas are never rendered.
- JSONL file writes respect backpressure.
- Artifact polling is replaced with event/subscription mechanisms where practical, but correctness precedes optimization.
- A worker runner must not hold unbounded in-memory output.

## 10. Compatibility boundaries

The integration validates behavior using both versions and capabilities.

Herdr requirements are checked against the installed schema. GSD-Pi requirements use exact patch manifests and source fingerprints until the generic seam exists upstream.

See:

- [`INTEGRATION_CONTRACT.md`](INTEGRATION_CONTRACT.md)
- [`UPSTREAM_MAINTENANCE.md`](UPSTREAM_MAINTENANCE.md)
- [`TESTING.md`](TESTING.md)
