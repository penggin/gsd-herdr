# Security

## 1. Security objective

`gsd-herdr` executes coding agents with the same user-level privileges as GSD-Pi and Herdr. It is not a sandbox. The security goal is therefore to prevent the integration layer from introducing avoidable credential exposure, command injection, path traversal, cross-pane authority confusion, duplicate execution, unsafe cleanup, or silent unmonitored work.

## 2. Trust boundaries

```text
User / shell environment
        │
        ▼
Main GSD process
        │ trusted to construct the child launch plan
        ▼
GSD extension and backend
        │ trusted integration code
        ▼
Secure launch artifacts
        │ local filesystem boundary
        ▼
Worker runner in Herdr pane
        │ process boundary
        ▼
GSD child and model/tool execution
```

Herdr's local CLI/socket API is another trust boundary. The integration assumes the active Herdr server belongs to the current user and validates its context/capabilities before use.

## 3. Threat model

### 3.1 Secrets exposed through command lines

Risk:

- API keys or environment values appear in process listings, pane output, logs, or crash reports.

Controls:

- launch argv contains the GSD command prepared by GSD-Pi, not a serialized full environment;
- environment values are written to a mode-`0600` one-time artifact;
- runner reads and deletes the environment artifact immediately;
- no environment dump is logged;
- no secrets are stored in Herdr metadata;
- support bundles exclude raw environment data.

### 3.2 Shell injection

Risk:

- task text, paths, model names, or arguments are concatenated into a shell command.

Controls:

- worker runner uses `spawn(executable, args, { shell: false })`;
- Herdr launches only the fixed worker-runner executable plus a validated spec path;
- user text is never used as a shell fragment;
- no `bash -lc` pipeline is used for subagent execution;
- paths and IDs are generated/validated before being passed to Herdr.

### 3.3 Parent-pane identity inherited by a child

Risk:

- a worker child inherits `HERDR_PANE_ID` from the main GSD process and reports status against the main pane.

Controls:

- strip all Herdr-managed keys from the parent launch environment;
- reapply only the runner process's actual worker-pane Herdr context;
- root GSD reporter refuses `GSD_SUBAGENT_CHILD=1`;
- root GSD reporter refuses non-TUI modes;
- worker authority is reported only by the worker runner.

Herdr-managed keys:

```text
HERDR_ENV
HERDR_SOCKET_PATH
HERDR_BIN_PATH
HERDR_WORKSPACE_ID
HERDR_TAB_ID
HERDR_PANE_ID
```

### 3.4 Raw model/tool data exposed in terminal output

Risk:

- raw JSONL includes prompts, tool arguments, model text deltas, usage data, or large payloads.

Controls:

- raw stdout is written to a protected JSONL artifact;
- `message_update` and token deltas are not rendered;
- tool arguments are converted through tool-specific bounded summaries;
- tool results are not rendered by default;
- stderr is captured and only a redacted bounded summary is shown;
- verbose mode does not bypass redaction or raw-JSON suppression.

### 3.5 Unsafe path cleanup

Risk:

- malicious/corrupt artifact paths or symlinks cause cleanup outside the integration state root.

Controls:

- state root is normalized and safety-checked during configuration loading;
- dispatch/child/attempt IDs use generated safe identifiers;
- every deletion resolves real paths and verifies containment;
- cleanup refuses unresolved, escaping, root-like, or user-home-wide paths;
- symlink ownership is verified before unlinking;
- recursive deletion targets only known integration-owned directories.

### 3.6 Fuzzy patching the wrong GSD-Pi version

Risk:

- a patch partially applies to a changed subagent implementation and alters execution semantics.

Controls:

- exact upstream version and source fingerprints;
- `git apply --check` or equivalent exact validation;
- no production fuzzy application or unconditional `--3way` fallback;
- focused patch tests and result-parity tests before activation;
- side-by-side build rather than in-place mutation;
- activation only after doctor/smoke success.

### 3.7 Duplicate execution

Risk:

- external launch succeeds, response is lost, and the integration starts a local or second external copy.

Controls:

- stable dispatch/child/attempt IDs;
- idempotency checks against durable attempt records;
- `started=true` forbids fallback;
- non-idempotent Herdr operations are reconciled before retry;
- once a runner/pane may exist, recovery searches by owned metadata rather than launching again;
- pane pool reservations are atomic within the extension process.

### 3.8 Stale or forged completion

Risk:

- a stale artifact from a prior attempt is mistaken for the current worker result.

Controls:

- artifact paths include dispatch, child, and attempt identity;
- exit artifact contains matching IDs and schema version;
- exit artifact is immutable after atomic publication;
- backend validates creation time/attempt ownership;
- late state sequence numbers cannot overwrite newer state;
- semantic success still requires GSD's existing result parser.

### 3.9 Unmonitored fallback

Risk:

- Herdr launch fails and GSD silently executes a child locally, allowing continued edits without a visible worker pane.

Controls:

- production default `required=true`, `fallback=error`;
- selected backend failures are explicit;
- local fallback is allowed only before any external side effect and only under explicit optional configuration;
- doctor warns when the patch/extension/backend is only partially active.

### 3.10 Malicious repository/project extension

Risk:

- project-local code may run with user privileges.

Controls:

- `gsd-herdr` should be installed globally from a trusted pinned release;
- do not auto-install project-controlled Herdr plugins or replacement runners;
- project-local config, if later supported, cannot replace executable paths or weaken security policy by default;
- installation records pin checksums and source revisions.

## 4. File permissions

Required defaults:

| Path/type | Mode |
|---|---:|
| configuration directory | `0700` |
| configuration file | `0600` |
| state root | `0700` |
| run/attempt directories | `0700` |
| launch/state/exit artifacts | `0600` |
| environment artifact | `0600` |
| stdout/stderr artifacts | `0600` |
| install metadata | `0600` or safe read-only equivalent |

The doctor reports weaker permissions and may repair integration-owned files with `--fix-safe`.

## 5. Environment handling

### 5.1 Capture

The GSD launch plan contains a process environment. The extension writes this to a one-time environment artifact only after:

- validating the target attempt directory;
- removing values that should be supplied by the worker pane;
- ensuring mode `0600` at creation.

### 5.2 Transfer

Only the file path is placed in the launch artifact. The environment content is never passed through Herdr metadata or pane output.

### 5.3 Consumption

The runner:

1. validates path containment;
2. opens the file without following unsafe ownership assumptions;
3. parses the expected schema;
4. stores values in memory;
5. deletes the artifact;
6. constructs the child environment;
7. clears references when the child has started.

### 5.4 Failure

When the environment artifact cannot be read or deleted:

- do not print its content;
- do not launch with an incomplete guessed environment;
- write a safe runner error/exit artifact;
- mark the pane failed;
- leave diagnostic path metadata only when safe.

## 6. Redaction

Built-in redaction should cover at least:

- environment variable names containing `KEY`, `TOKEN`, `SECRET`, `PASSWORD`, `COOKIE`, `AUTH`, `CREDENTIAL`;
- `Authorization`, `Proxy-Authorization`, and cookie headers;
- Bearer/basic tokens;
- common GitHub/OpenAI/Anthropic/provider token prefixes;
- URL query parameters named `token`, `key`, `secret`, `auth`, `signature`, `credential`;
- private key blocks;
- user-configured regular-expression patterns.

Replacement format:

```text
[REDACTED]
```

Rules:

- redaction occurs before truncation where possible so partial secrets are not preserved;
- failure to compile a user redaction pattern invalidates config;
- raw JSONL artifacts are not rewritten by the activity renderer; they are protected by filesystem permissions and retention policy;
- support bundles do not include raw JSONL by default.

## 7. Terminal rendering safety

Model/tool text may contain control characters or terminal escape sequences.

Controls:

- activity summaries strip or escape C0/C1 control characters except intentional newline/tab handling;
- OSC, CSI, DCS, APC, PM, and related terminal sequences are removed from untrusted summaries;
- line length and update frequency are bounded;
- file paths are displayed as text, not interpreted as clickable control payloads unless explicitly escaped by a safe renderer;
- pane titles/labels use restricted characters and bounded length.

## 8. Herdr API use

### 8.1 Context validation

Before mutating the current session, verify:

```text
HERDR_ENV=1
HERDR_SOCKET_PATH exists/reachable
HERDR_PANE_ID is present
current pane returned by Herdr matches the caller context
```

### 8.2 Explicit targets

Use explicit workspace/tab/pane IDs or `--current` only when the caller pane is intended. Never rely on an unrelated UI-focused pane.

### 8.3 Request classes

Idempotent operations may use bounded retry:

- state/metadata reports;
- reads;
- capability checks;
- release with sequence protection.

Non-idempotent operations require reconciliation before retry:

- tab creation;
- pane split;
- runner launch;
- pane close.

### 8.4 Ownership metadata

The integration only cleans or reuses panes carrying matching ownership metadata and durable records. It never closes arbitrary user panes.

## 9. Process and signal safety

- The runner owns the child process group on supported Unix platforms.
- Ctrl-C/SIGINT targets the runner/child attempt, not the main GSD process.
- Escalation timeouts are bounded and configurable.
- PID reuse is not trusted by itself; process evidence includes attempt identity and pane metadata.
- A successful signal send is not completion evidence.
- Exit artifacts are written after streams are flushed as far as safely possible.
- Forced termination is recorded distinctly from normal child exit.

## 10. State and log retention

- Successful/aborted data may be automatically expired according to config.
- Failed/orphaned/blocked evidence is retained by default.
- Storage pressure never deletes active or uncertain evidence automatically.
- Cleanup order and reason are logged.
- Cleanup has a dry-run mode.
- Raw JSONL may contain sensitive project information; documentation must communicate that clearly.

## 11. Installation security

- Pin released source/tag/commit and checksums.
- Review both upstream licenses and preserve required notices.
- Do not download/execute arbitrary project-provided scripts during normal installation.
- Build commands run only from the trusted `gsd-herdr` release and exact upstream source.
- Installation staging is on the same filesystem as the final target for atomic rename.
- Wrappers resolve only versioned installation records owned by the integration.
- A broken `current` symlink is an error, not a fallback to an arbitrary PATH binary.

## 12. Configuration security

- Unknown schema versions fail closed.
- `required=true` cannot be combined with `fallback=local`.
- `showRawJson=true` is rejected unless a controlled debug build explicitly supports it.
- Project-local configuration may not choose arbitrary runner executables.
- Environment overrides are allowlisted.
- Security settings cannot be weakened accidentally by an unknown field or typo.

## 13. Security tests

Required automated tests:

- task text with quotes, newlines, command substitutions, and shell metacharacters;
- malicious pane label/control sequences;
- symlink escape from attempt directory;
- `../../` artifact paths;
- environment artifact permissions and deletion;
- parent Herdr variable stripping;
- secret redaction in activity, stderr summary, logs, doctor output, and support bundle;
- duplicate launch after ambiguous Herdr response;
- stale exit artifact for prior attempt;
- late sequence update;
- active-pane cleanup refusal;
- fuzzy patch mismatch refusal;
- interrupted child process group without killing the parent;
- support bundle excludes raw JSON/environment by default.

## 14. Security incident behavior

When the integration detects a security invariant violation:

1. stop before launching new work where possible;
2. do not continue through local fallback;
3. record a stable error code and sanitized diagnostic;
4. preserve relevant non-secret evidence;
5. release only authority proven safe to release;
6. direct the user to `gsd-herdr doctor`;
7. never print the suspected secret value.

Examples:

```text
ARTIFACT_OUTSIDE_ROOT
ENVIRONMENT_UNAVAILABLE
UPSTREAM_PATCH_MISMATCH
PROTOCOL_MISMATCH
CONFIG_INVALID
```

## 15. Review requirements

Changes affecting any of the following require explicit security review and dedicated tests:

- process spawning;
- environment handling;
- path resolution/deletion;
- patch application;
- Herdr pane targeting;
- signal forwarding;
- redaction;
- support bundles;
- automatic recovery/adoption;
- fallback behavior.
