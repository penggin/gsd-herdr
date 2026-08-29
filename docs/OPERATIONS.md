# Operations

## 1. Operational goals

The stack must be safe to install, inspect, update, test, switch, roll back, and remove without modifying an active upstream installation in place.

Operational guarantees:

- installations are versioned and side-by-side;
- patch application targets an exact verified GSD-Pi source;
- the active version is selected through an atomic pointer/wrapper change;
- the previous known-good version remains available;
- Herdr plugins and GSD extensions are linked to the selected stack version;
- updates are validated before activation;
- uninstallation does not delete user project data or unrelated Herdr/GSD configuration;
- diagnostics are available without launching a real model request.

## 2. Command surface

Proposed CLI:

```text
gsd-herdr install
gsd-herdr doctor
gsd-herdr status
gsd-herdr update --check
gsd-herdr update --canary
gsd-herdr switch <stack-version>
gsd-herdr rollback
gsd-herdr cleanup [--dry-run]
gsd-herdr uninstall
```

Development-only commands:

```text
gsd-herdr dev link
gsd-herdr dev unlink
gsd-herdr patch check --gsd <ref>
gsd-herdr patch build --gsd <ref>
gsd-herdr contract check-herdr
gsd-herdr smoke
```

## 3. Filesystem layout

### 3.1 Installed program data

```text
~/.local/share/gsd-herdr/
├── versions/
│   ├── 0.1.0/
│   │   ├── bin/
│   │   ├── gsd-extension/
│   │   ├── herdr-plugin/
│   │   ├── worker-runner/
│   │   ├── patched-gsd/
│   │   ├── stack.lock.json
│   │   └── install.json
│   └── 0.2.0/
├── current -> versions/0.1.0
├── previous -> versions/0.0.9
└── installs.json
```

### 3.2 User configuration

```text
~/.config/gsd-herdr/
├── config.json
└── backups/
```

### 3.3 Runtime state

```text
~/.local/state/gsd-herdr/
├── sessions/
├── runs/
├── logs/
└── diagnostics/
```

### 3.4 User-facing wrappers

```text
~/.local/bin/gsd-herdr
~/.local/bin/gsd-herdr-canary   # optional
```

The installer does not replace the user's normal `gsd` command by default.

## 4. Installation flow

### 4.1 Preflight

`gsd-herdr install` performs:

1. detect OS and architecture;
2. verify supported platform;
3. verify Node.js version;
4. find Herdr and read its version;
5. find GSD-Pi and read its version/source;
6. inspect Herdr API schema/capabilities;
7. select a compatible stack entry;
8. select an exact GSD patch set;
9. ensure required directories can be created safely;
10. confirm no conflicting installation transaction is active.

The initial supported platform is macOS arm64.

### 4.2 Stage a new version

The installer creates a temporary staging directory under the same filesystem as the final version root.

```text
~/.local/share/gsd-herdr/.staging-<uuid>/
```

It then:

- copies/builds the `gsd-herdr` packages;
- obtains the exact GSD-Pi source or release package required by the compatibility manifest;
- verifies source fingerprints;
- runs `git apply --check` or an equivalent exact patch validation;
- applies the selected patch set;
- builds or overlays GSD-Pi according to the M0 spike decision;
- writes an immutable stack lock;
- runs focused tests and a smoke test;
- records checksums.

Only after all checks succeed is the staging directory atomically renamed into `versions/<version>`.

### 4.3 Install the GSD extension

Preferred representation:

```text
~/.gsd/agent/extensions/gsd-herdr
  -> ~/.local/share/gsd-herdr/current/gsd-extension
```

The installer must preserve any existing non-integration file at that destination. If the path already exists and is not owned by `gsd-herdr`, installation stops with a conflict error.

Ownership is recorded through both:

- a marker file in the installed extension directory;
- `installs.json` containing the resolved source/target pair.

### 4.4 Install the Herdr plugin

Development:

```bash
herdr plugin link /path/to/gsd-herdr/packages/herdr-plugin
```

Release installation links or installs the selected versioned directory. The installer verifies the plugin ID and source before replacing an existing integration-owned link.

### 4.5 Install wrapper

`~/.local/bin/gsd-herdr` resolves the current stack and launches the versioned patched GSD with the selected extension/runtime paths.

Responsibilities:

- fail if `current` is broken;
- set only required integration paths/variables;
- preserve the user's shell environment;
- not print secrets;
- use `exec` so signals and exit codes propagate;
- require Herdr context when configuration says monitoring is required.

### 4.6 Smoke test

The smoke test does not call a paid model provider.

It creates an isolated Herdr test context and uses a fake GSD child that emits deterministic JSONL.

Validation:

- root extension loads;
- backend resolves;
- worker pane is created;
- runner starts;
- pane shows readable activity;
- pane does not show raw JSON;
- raw JSONL artifact exists;
- state and exit artifacts are valid;
- interrupt path works in a second fixture;
- test panes/session are cleaned up.

### 4.7 Activation

Activation updates pointers in this order:

1. write `previous` target from the current valid target;
2. atomically replace `current` symlink/pointer;
3. reconcile GSD extension link;
4. reconcile Herdr plugin link;
5. run `doctor` against the active target.

If post-activation doctor fails, activation automatically restores the prior pointers and reports both the primary failure and rollback result.

## 5. `doctor`

`gsd-herdr doctor` is the primary support command.

Checks:

### Stack

- selected stack version;
- stack lock validity;
- checksums;
- current/previous pointers;
- wrapper resolution.

### GSD-Pi

- installed version and commit/source identity;
- patch manifest match;
- generic backend protocol marker;
- GSD extension discovery;
- extension manifest validity;
- protocol compatibility;
- runtime entrypoint readability.

### Herdr

- binary existence/version;
- `HERDR_ENV` and pane variables when run inside a pane;
- socket reachability;
- required methods in API schema;
- plugin registration and enabled state;
- current pane access;
- ability to report and release test metadata/state when safe.

### Filesystem/security

- config readability;
- config validity;
- state-root ownership and permissions;
- no unsafe symlink/path escapes;
- installed extension/plugin ownership;
- sufficient disk space;
- stale one-time environment artifacts.

### Runtime

- active sessions/workers;
- stale heartbeats;
- orphan records;
- missing panes;
- stale agent authority;
- expired retained runs.

Output modes:

```bash
gsd-herdr doctor
gsd-herdr doctor --json
gsd-herdr doctor --fix-safe
```

`--fix-safe` may only perform idempotent, non-destructive actions such as correcting integration-owned file permissions or releasing proven stale integration authority. It must not close active panes or delete failed/orphan evidence.

## 6. `status`

`gsd-herdr status` reports the installed stack and runtime state without mutation.

Example:

```text
Stack 0.1.0 · healthy
GSD-Pi 1.16.2 + external-backend-v1
Herdr 0.8.2 · required capabilities present

Main sessions: 1
Workers: 3 running, 1 failed, 0 orphaned
Storage: 182 MiB / 2048 MiB soft cap

pengbot_monorepo
  main    w1:p1   working   M04/S02/T03
  falcon  w1:p5   working   read STATE.md
  cedar   w1:p6   completed exit 0
  harbor  w1:p7   failed    provider 503
```

## 7. Update flow

### 7.1 Check only

```bash
gsd-herdr update --check
```

The command discovers:

- latest `gsd-herdr` stack release;
- latest supported Herdr stable version;
- latest supported GSD-Pi stable version;
- API capability compatibility;
- patch applicability;
- migration requirements.

It makes no changes.

### 7.2 Canary

```bash
gsd-herdr update --canary
```

A canary installation:

- is staged side-by-side;
- does not change `current`;
- installs a `gsd-herdr-canary` wrapper;
- may use a separate extension/plugin identifier or explicit launch path to avoid interfering with stable;
- uses a separate runtime namespace when simultaneous testing is required;
- runs the full smoke suite.

Canary output includes an exact command to test a disposable project/session.

### 7.3 Promote

```bash
gsd-herdr switch <version>
```

Promotion requires:

- successful installation record;
- successful smoke result;
- compatible current config or completed migration;
- no active installation lock;
- confirmation when active workers exist.

Active workers are never silently migrated between stack versions.

## 8. Rollback

```bash
gsd-herdr rollback
```

Rollback changes the active version pointers and integration links. It does not rewrite historical run artifacts.

Rules:

- active workers trigger a warning and require explicit confirmation or `--defer`;
- config schema must be readable by the target version;
- if migration occurred, use the retained backup when required;
- previous patched GSD and runner binaries remain available;
- rollback itself is verified with doctor.

`rollback --force` is reserved for a broken active wrapper and may only operate on known installation records.

## 9. Cleanup

```bash
gsd-herdr cleanup --dry-run
gsd-herdr cleanup
gsd-herdr cleanup --failed <run-id>
```

Automatic eligibility order:

1. expired successful runs;
2. expired aborted runs;
3. old debug diagnostics;
4. unused old installed stack versions, subject to retention count.

Never automatically delete:

- active runs;
- blocked workers;
- failed runs under indefinite retention;
- orphaned runs under indefinite retention;
- current or previous stack version;
- state outside the integration root.

A dry run lists paths, sizes, reason, and ownership evidence.

## 10. Uninstall

```bash
gsd-herdr uninstall
```

Default behavior:

- remove integration-owned wrappers;
- unlink the GSD extension;
- unlink/uninstall the Herdr plugin;
- remove installed stack program versions;
- preserve config backups and runtime evidence;
- leave official Herdr and official GSD-Pi installations untouched.

Optional flags:

```text
--purge-config
--purge-completed-state
--purge-all-state
```

`--purge-all-state` requires explicit confirmation and refuses while active worker processes are detected unless `--force` is supplied.

## 11. Runtime operations

### Start

```bash
herdr
```

Inside a Herdr pane:

```bash
gsd-herdr
```

When `required=true`, launching outside Herdr fails with a clear message.

### Detach and reattach

Use normal Herdr detach/reattach behavior. The stack does not terminate workers on client detachment.

### Inspect workers

From GSD:

```text
/herdr-status
/herdr-doctor
/herdr-cleanup
```

From the shell/plugin:

```bash
gsd-herdr status
herdr plugin action invoke penggin.gsd-herdr.status
```

### Interrupt

The normal GSD cancellation path remains authoritative. Manual pane Ctrl-C may also interrupt the worker, but GSD must detect the resulting exit artifact/state rather than waiting indefinitely.

### Close a pane

Closing an active worker pane is treated as an abnormal worker loss. The parent attempt fails explicitly unless the child had already published a final exit artifact.

## 12. Installation transaction safety

A lock file prevents concurrent install/update/switch/rollback/uninstall operations.

Lock metadata:

```json
{
  "pid": 12345,
  "startedAt": "2026-08-29T...",
  "command": "update --canary",
  "host": "..."
}
```

Stale-lock recovery requires:

- process not alive;
- lock older than a bounded threshold;
- no staging rename in progress;
- explicit diagnostic output.

## 13. Logging

Operational logs live under:

```text
~/.local/state/gsd-herdr/logs/
```

Logs contain:

- operation ID;
- stack versions;
- step names;
- status and timings;
- sanitized command summaries;
- error codes;
- safe artifact paths.

They do not contain:

- environment dumps;
- API keys;
- complete prompts;
- raw tool results;
- authorization headers.

## 14. Support bundle

A future command may generate a sanitized support bundle:

```bash
gsd-herdr doctor --bundle /path/to/bundle.zip
```

It may include:

- versions and checksums;
- config with secrets/paths redacted;
- doctor JSON;
- protocol and capability report;
- selected sanitized logs;
- directory/file metadata;
- no raw JSONL or project file contents unless explicitly approved.

This command is not required for the first MVP but should shape logging and diagnostics from the start.
