# Integration Contract

## 1. Purpose

This document defines the versioned boundaries between:

- the minimal GSD-Pi patch;
- the `gsd-herdr` GSD extension;
- the Herdr backend;
- the worker runner;
- durable artifacts;
- the Herdr operations plugin.

The contracts are designed to fail explicitly when versions or capabilities do not match. Internal implementation details may change without a protocol bump; serialized or cross-package behavior may not.

## 2. Versioning rules

The first protocol version is `1`.

A protocol version changes when any of the following becomes incompatible:

- required request fields;
- callback behavior;
- state names or semantics;
- artifact shape;
- path ownership or cleanup guarantees;
- sequence-number interpretation;
- cancellation behavior;
- completion evidence.

Additive optional fields do not require a major protocol bump when older consumers can safely ignore them.

Every cross-component payload includes:

```ts
interface VersionedPayload {
  protocolVersion: 1;
}
```

Artifacts additionally include a schema name:

```ts
interface ArtifactHeader {
  schema: "gsd-herdr/launch" | "gsd-herdr/state" | "gsd-herdr/exit" | "gsd-herdr/run";
  schemaVersion: 1;
}
```

## 3. Backend discovery contract

### 3.1 Event channel

```ts
export const EXTERNAL_SUBAGENT_BACKEND_CHANNEL =
  "gsd:subagent-backend:resolve:v1";
```

The channel name contains the protocol version so incompatible providers do not accidentally answer.

### 3.2 Request

```ts
export interface ResolveExternalSubagentBackendRequestV1 {
  protocolVersion: 1;

  context: {
    cwd: string;
    hasUI: boolean;
    parentSessionId?: string;
    parentSessionFile?: string;
    dispatchMode: "single" | "parallel" | "chain";
  };

  respond(backend: ExternalSubagentBackendV1): void;
}
```

Requirements:

- `respond()` must be called synchronously from the event handler.
- Discovery must not create panes, files, or processes.
- A provider may decline by doing nothing.
- A provider must not throw to signal unavailability.
- Invalid providers are ignored and reported through diagnostics.

### 3.3 Backend descriptor

```ts
export interface ExternalSubagentBackendV1 {
  id: string;
  protocolVersion: 1;
  priority: number;
  fallbackPolicy: "local" | "error";

  execute(
    request: ExternalSubagentExecutionRequestV1,
  ): Promise<ExternalSubagentExecutionResultV1>;
}
```

Selection order:

1. highest `priority`;
2. lexical `id` for deterministic ties.

The selected backend ID is recorded in the run metadata.

## 4. External execution request

```ts
export interface ExternalSubagentExecutionRequestV1 {
  protocolVersion: 1;

  dispatch: {
    dispatchId: string;
    childId: string;
    mode: "single" | "parallel" | "chain";
    index: number;
    total: number;
    attempt: number;
    step?: number;
    background: boolean;
  };

  display: {
    agent: string;
    trackingName?: string;
    task: string;
    model?: string;
    thinking?: string;
  };

  launch: {
    executable: string;
    args: string[];
    cwd: string;
    env: NodeJS.ProcessEnv;
  };

  signal?: AbortSignal;

  onStdoutLine(line: string): void;
  onStderrChunk(chunk: string): void;
  onActivity?(activity: ExternalSubagentActivityV1): void;
}
```

### 4.1 Identity

- `dispatchId` identifies one top-level subagent-tool dispatch.
- `childId` identifies one logical child within the dispatch.
- `attempt` changes for retries of the same child.
- `index` and `total` describe the logical batch, not currently running slots.
- A chain step may either use a stable child ID plus `step`, or a unique child ID per step. Version 1 uses a stable child ID for one chain execution and includes `step`.

### 4.2 Launch command

The backend must treat `executable` and `args` as an argv vector. It must not concatenate them into an unescaped shell command.

The launch plan is prepared by GSD-Pi and remains authoritative for:

- `--mode json`;
- model and thinking overrides;
- tools;
- system prompt file;
- fresh/fork session arguments;
- task text;
- project/runtime contract variables.

The backend may remove and replace Herdr-managed environment variables, but it must not otherwise reinterpret GSD launch semantics.

### 4.3 Output callbacks

`onStdoutLine()` requirements:

- called once for every complete non-empty stdout JSONL line;
- called in child-output order;
- never called with the trailing newline;
- malformed lines are still delivered so the existing GSD parser can apply its behavior;
- the backend must flush a final buffered line at EOF.

`onStderrChunk()` requirements:

- called with stderr text in observed order;
- may be called with arbitrary chunk boundaries;
- must not omit captured stderr;
- must not expose stderr to the pane unless display policy permits it.

Callbacks are in-process and are not serialized into launch artifacts.

### 4.4 Cancellation

The backend must observe `signal`.

If the signal is already aborted before process launch, the backend must return an aborted result without launching a worker.

If cancellation occurs after launch:

1. request graceful interruption;
2. observe completion evidence;
3. escalate according to configured timeouts;
4. return only after a terminal state or a bounded backend error.

The backend must never start a local duplicate after an external child may have begun execution.

## 5. External execution result

```ts
export interface ExternalSubagentExecutionResultV1 {
  protocolVersion: 1;

  handled: boolean;
  started: boolean;
  exitCode: number;
  aborted: boolean;
  stderr: string;

  external: {
    backend: string;
    workspaceId?: string;
    tabId?: string;
    paneId?: string;
    artifactDir?: string;
    attemptId?: string;
  };
}
```

Semantics:

- `handled=false` is only valid before any pane/process side effect. It permits fallback when policy allows.
- `started=true` means an external worker may have performed work; fallback is forbidden.
- `exitCode` is the child process exit code or a normalized nonzero backend failure code.
- `aborted=true` records user/parent cancellation even if the child exits with another code.
- `stderr` contains the complete captured stderr within configured retention/cap policy.
- `external` is observational metadata and must not alter GSD's semantic result parsing.

Recommended normalized backend codes when no child exit code exists:

| Code | Meaning |
|---:|---|
| `120` | backend unavailable before launch |
| `121` | pane allocation failed |
| `122` | runner launch failed |
| `123` | worker pane disappeared |
| `124` | worker completion timed out |
| `125` | protocol/artifact mismatch |
| `130` | interrupted |

These codes are integration-local and do not replace an observed child exit code.

## 6. Activity contract

Activity is optional diagnostic information and does not determine the GSD result.

```ts
export interface ExternalSubagentActivityV1 {
  protocolVersion: 1;
  timestamp: string;
  kind:
    | "starting"
    | "working"
    | "tool-start"
    | "tool-end"
    | "retrying"
    | "blocked"
    | "completed"
    | "failed"
    | "aborted"
    | "orphaned";
  summary: string;
  toolName?: string;
  isError?: boolean;
}
```

Rules:

- `summary` is human-readable, bounded, and redacted.
- No raw model token delta is represented.
- No complete tool result or environment is included.
- Consumers must tolerate unknown future `kind` values by treating them as informational.

## 7. Launch artifact

Path:

```text
<state-root>/runs/<dispatch-id>/children/<child-id>/attempts/<attempt>/launch.json
```

Schema:

```ts
export interface LaunchArtifactV1 {
  schema: "gsd-herdr/launch";
  schemaVersion: 1;

  createdAt: string;
  dispatchId: string;
  childId: string;
  attempt: number;

  mode: "single" | "parallel" | "chain";
  index: number;
  total: number;
  step?: number;
  background: boolean;

  agent: string;
  trackingName?: string;
  model?: string;
  thinking?: string;
  taskPreview: string;
  taskHash: string;

  executable: string;
  args: string[];
  cwd: string;

  environmentPath: string;
  stdoutPath: string;
  stderrPath: string;
  statePath: string;
  heartbeatPath: string;
  exitPath: string;
}
```

Security rules:

- no secret environment values in `launch.json`;
- full task text may exist in the GSD argv because GSD created it, but logs and metadata use only `taskPreview` and `taskHash`;
- file mode `0600`;
- parent directory mode `0700`;
- all paths must resolve inside the configured state root except `executable` and `cwd`;
- runner validates every path before use.

## 8. Environment artifact

Path is referenced by `environmentPath`.

```ts
export interface EnvironmentArtifactV1 {
  schema: "gsd-herdr/environment";
  schemaVersion: 1;
  values: Record<string, string>;
}
```

Rules:

1. written with `0600` mode;
2. never printed or included in Herdr metadata;
3. read once by the runner;
4. deleted immediately after successful read;
5. parent Herdr-managed keys are removed;
6. worker-pane Herdr-managed values are reapplied from the runner's environment;
7. missing/deletion failure is reported but secret values are never echoed.

Herdr-managed keys:

```text
HERDR_ENV
HERDR_SOCKET_PATH
HERDR_BIN_PATH
HERDR_WORKSPACE_ID
HERDR_TAB_ID
HERDR_PANE_ID
```

## 9. Worker state artifact

```ts
export interface WorkerStateArtifactV1 {
  schema: "gsd-herdr/state";
  schemaVersion: 1;

  updatedAt: string;
  sequence: number;
  dispatchId: string;
  childId: string;
  attempt: number;

  state:
    | "reserved"
    | "starting"
    | "working"
    | "retrying"
    | "blocked"
    | "completed"
    | "failed"
    | "aborted"
    | "orphaned";

  summary?: string;
  lastActivity?: ExternalSubagentActivityV1;

  parent: {
    sessionId?: string;
    sessionFile?: string;
    paneId?: string;
    heartbeatAt?: string;
  };

  worker: {
    workspaceId?: string;
    tabId?: string;
    paneId?: string;
    runnerPid?: number;
    childPid?: number;
  };
}
```

Writes are atomic: write a temporary sibling file, fsync when required by policy, then rename.

## 10. Heartbeat artifact

The heartbeat is intentionally small to make frequent updates inexpensive.

```ts
export interface HeartbeatArtifactV1 {
  schema: "gsd-herdr/heartbeat";
  schemaVersion: 1;
  at: string;
  sequence: number;
  runnerPid: number;
  childPid?: number;
}
```

Default interval: 5 seconds.

A stale heartbeat alone does not prove failure. Reconciliation combines:

- heartbeat age;
- pane existence;
- foreground process information;
- exit artifact presence;
- parent session state.

## 11. Exit artifact

```ts
export interface ExitArtifactV1 {
  schema: "gsd-herdr/exit";
  schemaVersion: 1;

  completedAt: string;
  dispatchId: string;
  childId: string;
  attempt: number;

  exitCode: number;
  signal?: string;
  aborted: boolean;

  evidence:
    | "child-exit"
    | "runner-error"
    | "pane-lost"
    | "protocol-error"
    | "forced-termination";

  stdoutBytes: number;
  stderrBytes: number;
  finalState: "completed" | "failed" | "aborted" | "orphaned";
  error?: string;
}
```

The artifact is final and immutable after atomic publication. Reconciliation must not rewrite it.

## 12. Pane metadata contract

The extension/runner may report metadata under a namespaced key set:

```json
{
  "gsdHerdr.protocol": 1,
  "gsdHerdr.role": "main|worker",
  "gsdHerdr.dispatchId": "...",
  "gsdHerdr.childId": "...",
  "gsdHerdr.attempt": 1,
  "gsdHerdr.trackingName": "falcon",
  "gsdHerdr.agent": "scout",
  "gsdHerdr.state": "working",
  "gsdHerdr.taskHash": "sha256:..."
}
```

Metadata must not contain:

- complete prompts;
- API keys or tokens;
- arbitrary tool output;
- full environment values;
- user file contents.

## 13. Required Herdr capabilities

Protocol v1 requires public support equivalent to:

```text
session.snapshot
workspace.get or workspace.list
tab.create
tab.list
pane.split or layout.apply
pane.run or pane input sufficient to start the runner
pane.get / pane.list
pane.read
pane.send_keys
pane.process_info
pane.report_agent
pane.report_agent_session
pane.report_metadata
pane.release_agent
pane.close
```

Capability groups:

### Required for MVP

- identify current context;
- create/reuse tab and panes;
- run worker command;
- interrupt worker;
- report/release state;
- inspect pane existence/process.

### Optional improvements

- declarative `layout.apply`;
- event subscriptions;
- richer metadata;
- agent view APIs;
- direct focus by reported agent name.

The doctor checks the installed API schema rather than trusting a version number alone.

## 14. Error handling

Errors are classified as:

```ts
export type GsdHerdrErrorCode =
  | "CONFIG_INVALID"
  | "PROTOCOL_MISMATCH"
  | "HERDR_NOT_ACTIVE"
  | "HERDR_CAPABILITY_MISSING"
  | "PANE_ALLOCATION_FAILED"
  | "RUNNER_LAUNCH_FAILED"
  | "WORKER_PANE_LOST"
  | "WORKER_TIMEOUT"
  | "ARTIFACT_INVALID"
  | "ARTIFACT_OUTSIDE_ROOT"
  | "ENVIRONMENT_UNAVAILABLE"
  | "CANCELLED"
  | "UPSTREAM_PATCH_MISMATCH"
  | "INTERNAL_ERROR";
```

User-facing errors include:

- stable code;
- concise message;
- affected dispatch/child where available;
- log or artifact path where safe;
- suggested recovery action.

They never include secret values.

## 15. Compatibility behavior

### Patch newer than extension

Backend discovery receives an unsupported channel/version and finds no provider. In required mode, startup/doctor should catch this before dispatch.

### Extension newer than patch

The extension may report the main session, but backend execution remains unavailable. `/herdr-doctor` must identify this partial state.

### Runner mismatch

The runner refuses the launch artifact before spawning the child and writes a protocol-error exit artifact when possible.

### Herdr capability mismatch

The backend fails before creating a worker when required methods are missing. It must not partially allocate a pool and silently fall back.

## 16. Contract test requirements

Every protocol implementation must have tests covering:

- valid round-trip;
- missing required fields;
- unknown optional fields;
- unsupported protocol version;
- duplicate/late state sequence;
- cancellation before and after launch;
- malformed JSONL;
- missing pane;
- artifact path escaping state root;
- secret redaction;
- exit artifact immutability;
- result parity with local GSD execution.
