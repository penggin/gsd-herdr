# Configuration

## 1. Status

This document defines the proposed configuration contract for protocol/configuration schema version `1`. Names and defaults may change during M0/M1 technical validation, but behavior changes must be reflected here and in `PLANNING.md`.

The integration owns a separate configuration file rather than modifying GSD-Pi's `PREFERENCES.md` schema during the first implementation.

Default path:

```text
~/.config/gsd-herdr/config.json
```

Optional override:

```text
GSD_HERDR_CONFIG=/absolute/path/config.json
```

## 2. Complete proposed configuration

```json
{
  "version": 1,
  "enabled": true,
  "required": true,
  "fallback": "error",

  "layout": {
    "mode": "session-pool",
    "workerTab": true,
    "workerTabLabel": "GSD Workers",
    "maxPanes": 4,
    "preserveMainFocus": true,
    "reuseChainPane": true,
    "reuseRetryPane": true
  },

  "display": {
    "level": "activity",
    "showModel": true,
    "showThinking": false,
    "showTaskPreview": true,
    "showElapsed": true,
    "showFinalSummary": true,
    "showRawJson": false,
    "showLiveStderr": false,
    "maxTaskPreviewChars": 120,
    "maxCommandChars": 120,
    "maxSummaryChars": 240,
    "maxPaneUpdatesPerSecond": 5,
    "maxStateReportsPerSecond": 2
  },

  "retention": {
    "successMinutes": 10,
    "abortedMinutes": 10,
    "failureMinutes": null,
    "orphanMinutes": null,
    "logHours": 72,
    "maxRunStorageMb": 2048
  },

  "runtime": {
    "stateDir": "~/.local/state/gsd-herdr",
    "heartbeatMs": 5000,
    "parentHeartbeatTimeoutMs": 20000,
    "completionTimeoutMs": 1800000,
    "interruptGraceMs": 5000,
    "terminateGraceMs": 5000,
    "socketRequestTimeoutMs": 1000,
    "socketRetryCount": 1
  },

  "recovery": {
    "onParentLoss": "keep-running",
    "onPaneLoss": "fail",
    "reconcileOnStartup": true,
    "releaseStaleAuthority": true,
    "autoCloseExpiredSuccess": false
  },

  "security": {
    "redact": true,
    "deleteEnvironmentAfterRead": true,
    "allowTaskPreview": true,
    "additionalSecretPatterns": []
  },

  "diagnostics": {
    "logLevel": "info",
    "writeDebugEvents": false,
    "includeToolArguments": "summary"
  }
}
```

## 3. Top-level options

### `version`

Configuration schema version. Required.

```json
"version": 1
```

An unsupported version is a startup/doctor error. The integration must not guess or silently reinterpret unknown schemas.

### `enabled`

Master switch.

```json
"enabled": true
```

When false:

- main GSD state is not reported;
- the Herdr external backend does not register;
- worker panes are not created;
- installed plugin operations may still show retained historical state.

### `required`

Declares that monitored Herdr execution is a correctness requirement.

```json
"required": true
```

When true, a missing or failed backend must fail the dispatch visibly. It must not create an invisible local child.

### `fallback`

```json
"fallback": "error"
```

Allowed values:

- `"error"` — fail if the selected backend cannot safely start or complete.
- `"local"` — allow local GSD execution only if the backend declines or fails **before any external process may have started**.

`required: true` implies `fallback: "error"`. A contradictory config is invalid.

## 4. Layout

### `mode`

Initial allowed value:

```json
"mode": "session-pool"
```

One persistent worker pool is associated with one root GSD session.

Reserved future values:

- `"dispatch-tab"` — one tab per dispatch;
- `"current-tab"` — workers beside the main pane.

Unsupported values are rejected.

### `workerTab`

Creates or reuses a dedicated worker tab. Default: `true`.

### `workerTabLabel`

Base label for worker tabs. The implementation may append a project or session suffix when needed for uniqueness.

### `maxPanes`

Default: `4`.

Constraints:

```text
minimum: 1
maximum in schema v1: 8
recommended: 4
```

This is visible capacity, not permission to exceed GSD-Pi's orchestration concurrency.

### `preserveMainFocus`

When true, background worker allocation and launch must not move user focus from the main GSD pane.

### `reuseChainPane`

Keeps sequential chain steps in the same pane when possible.

### `reuseRetryPane`

Keeps retries of one logical child in the same pane and appends an attempt separator.

## 5. Display

### `level`

Allowed values:

| Value | Pane output |
|---|---|
| `status` | starting, working, retrying, blocked, completed, failed |
| `activity` | status plus concise tool starts and selected lifecycle events |
| `verbose` | activity plus bounded final summaries and selected tool completions |

Default: `activity`.

`verbose` still does not render token deltas or raw JSON.

### `showRawJson`

Default and recommended value: `false`.

Schema v1 may reject `true` in production builds. The option exists only to make the safety rule explicit and support controlled debugging if later implemented.

### `showLiveStderr`

When false, stderr is captured in an artifact and surfaced only as a final bounded error summary. When true, redacted stderr may be mirrored live.

Default: `false`.

### Preview limits

- `maxTaskPreviewChars`: default 120.
- `maxCommandChars`: default 120.
- `maxSummaryChars`: default 240.

All limits apply after redaction and before terminal output or metadata reporting.

### Rate limits

- `maxPaneUpdatesPerSecond`: default 5.
- `maxStateReportsPerSecond`: default 2.

Identical consecutive activities are deduplicated regardless of limits.

## 6. Retention

A JSON `null` means no automatic age-based deletion for that category.

### `successMinutes`

How long a successful pane remains visible before becoming eligible for reuse. Default: 10.

### `abortedMinutes`

How long an aborted pane remains visible. Default: 10.

### `failureMinutes`

Default: `null`. Failed panes remain until manual cleanup.

### `orphanMinutes`

Default: `null`. Orphan evidence is retained until resolved or manually cleaned.

### `logHours`

Default: 72. Applies to completed runs not otherwise retained by failure/orphan policy.

### `maxRunStorageMb`

Soft cap for integration-owned run artifacts. Cleanup removes oldest eligible successful/aborted data first and never silently removes active, blocked, failed, or orphaned evidence.

## 7. Runtime

### `stateDir`

Default:

```text
~/.local/state/gsd-herdr
```

Requirements:

- absolute after expansion;
- owned/writable by the current user;
- integration can establish `0700` permissions;
- must not resolve to `/`, home itself, or another broad system directory;
- artifact cleanup refuses paths outside this root.

### Heartbeats

- `heartbeatMs`: worker heartbeat interval, default 5000.
- `parentHeartbeatTimeoutMs`: threshold for considering the parent stale, default 20000.

Heartbeat expiry triggers investigation/reconciliation, not immediate success/failure assumptions.

### `completionTimeoutMs`

Maximum time the backend waits for a terminal worker result before entering timeout/cancellation handling. Default: 30 minutes.

Long-running workflows may override this.

### Signal grace periods

- `interruptGraceMs`: time after SIGINT/Ctrl-C before SIGTERM.
- `terminateGraceMs`: time after SIGTERM before SIGKILL.

### Socket settings

- `socketRequestTimeoutMs`: timeout per Herdr request attempt.
- `socketRetryCount`: bounded retry count for idempotent state/report operations.

Non-idempotent pane creation must not be blindly retried without checking whether the first request succeeded.

## 8. Recovery

### `onParentLoss`

Allowed values:

- `"keep-running"` — default; mark worker orphaned and preserve work/evidence.
- `"interrupt"` — request graceful cancellation after parent timeout.
- `"terminate"` — terminate after configured grace periods.

The first stable release defaults to `keep-running` because a false-positive parent-loss decision must not destroy work.

### `onPaneLoss`

Schema v1 allowed value:

```json
"onPaneLoss": "fail"
```

If a worker pane disappears without an exit artifact, the attempt fails explicitly.

### `reconcileOnStartup`

Enables Herdr plugin startup reconciliation.

### `releaseStaleAuthority`

Releases integration-owned pane agent authority when no valid owner remains.

### `autoCloseExpiredSuccess`

Default: `false`. Expired successful panes become reusable but are not necessarily closed unless pool maintenance requires it.

## 9. Security

### `redact`

Must remain true in supported production configurations.

### `deleteEnvironmentAfterRead`

Default: true. The runner deletes the one-time environment artifact immediately after reading it.

### `allowTaskPreview`

When false, pane and metadata show only task hash/agent identity, not a text preview.

### `additionalSecretPatterns`

Optional regular-expression strings applied in addition to built-in redaction. Invalid expressions fail config validation.

## 10. Diagnostics

### `logLevel`

Allowed:

```text
error
warn
info
debug
trace
```

Default: `info`.

`debug` and `trace` still must not log secret values or the full environment.

### `writeDebugEvents`

When enabled, sanitized protocol/state transitions may be written under the state root. Raw JSONL already exists separately and is not duplicated.

### `includeToolArguments`

Allowed values:

- `none`;
- `summary` — default; safe, tool-specific bounded summaries;
- `redacted` — bounded redacted serialized arguments.

## 11. Environment overrides

Proposed overrides:

```text
GSD_HERDR_DISABLE=1
GSD_HERDR_REQUIRED=1
GSD_HERDR_FALLBACK=error|local
GSD_HERDR_STATE_DIR=/absolute/path
GSD_HERDR_MAX_PANES=4
GSD_HERDR_DISPLAY=status|activity|verbose
GSD_HERDR_LOG_LEVEL=error|warn|info|debug|trace
```

Precedence, highest first:

1. safety override `GSD_HERDR_DISABLE=1`;
2. explicit supported environment overrides;
3. configuration file;
4. built-in defaults.

Environment variables not listed here do not change behavior.

## 12. Per-project configuration

The first implementation uses global configuration only. A later additive feature may support:

```text
<project>/.gsd/gsd-herdr.json
```

When added, project config may narrow or override layout/display behavior but must not weaken globally enforced security or required-monitoring policy without an explicit global permission.

## 13. Validation behavior

Configuration loading returns either a fully normalized config or a structured error. It does not partially apply invalid input.

Examples of invalid configuration:

- unknown schema version;
- `required=true` with `fallback=local`;
- `maxPanes=0`;
- negative timeout;
- unsafe state root;
- malformed secret pattern;
- `showRawJson=true` when unsupported;
- unknown enum value.

The doctor reports:

```text
CONFIG_INVALID
path: ~/.config/gsd-herdr/config.json
field: layout.maxPanes
message: expected integer between 1 and 8
```

## 14. Migration policy

Migrations are explicit and reversible.

- The CLI never rewrites config without showing or recording the change.
- A migrated file is written atomically.
- The previous file is retained as a timestamped backup.
- Unknown fields are preserved when safe.
- Breaking schema changes require a new `version`.
- Rollback must continue to understand at least the immediately previous schema.
