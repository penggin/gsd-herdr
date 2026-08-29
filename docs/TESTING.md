# Testing Strategy

## 1. Quality objective

The integration is correct only when an externally executed Herdr worker produces the same semantic GSD result as the existing local subagent path while adding reliable observability and persistence.

Testing therefore has two independent dimensions:

1. **semantic parity** — GSD result, retry, cancellation, session, isolation, and usage behavior remain correct;
2. **runtime observability** — Herdr panes, state, artifacts, and recovery accurately represent the worker.

Passing only one dimension is insufficient.

## 2. Test layers

```text
static validation
      ↓
unit tests
      ↓
contract tests
      ↓
integration tests with fake Herdr/GSD
      ↓
patch compatibility tests against real GSD source
      ↓
Herdr API tests against real Herdr binary
      ↓
end-to-end tests in an isolated Herdr session
      ↓
manual canary validation
```

## 3. Static validation

Required on every change:

- TypeScript type checking;
- formatting/linting;
- package dependency boundary checks;
- JSON/TOML schema validation;
- patch manifest/fingerprint validation;
- no generated artifact drift;
- secret-pattern checks for committed fixtures;
- license and attribution checks when upstream code is incorporated.

No package may import implementation code in the reverse direction of the documented dependency graph.

## 4. Unit tests

### 4.1 Protocol package

Test:

- valid backend descriptor;
- invalid backend ID/version/priority;
- launch request validation;
- result validation;
- state transition validation;
- artifact schema validation;
- unknown additive fields;
- unsupported version failure;
- deterministic backend selection;
- sequence ordering and stale update rejection;
- normalized error codes.

### 4.2 Herdr client

Use a fake CLI/socket endpoint.

Test:

- schema/capability parsing;
- missing required methods;
- current-context resolution;
- normalized success/error responses;
- timeout and bounded retry;
- idempotent state reporting;
- non-idempotent operation ambiguity;
- explicit pane targeting;
- response ID extraction;
- invalid/malformed JSON;
- process exit without response.

### 4.3 GSD extension root reporter

Test:

- disabled config;
- non-Herdr environment;
- missing pane/socket context;
- TUI session accepted;
- JSON/RPC/print modes rejected;
- `GSD_SUBAGENT_CHILD=1` rejected;
- `agent_start` → working;
- settled/end → idle;
- blocked count nesting;
- milestone/slice/task label updates;
- extension reload deduplication;
- shutdown releases authority;
- socket failure does not crash GSD.

### 4.4 Backend provider

Test:

- discovery responds synchronously;
- discovery has no side effects;
- required/optional fallback policy;
- capability failure before launch;
- pane slot allocation;
- launch artifact creation;
- callback ordering;
- JSONL line relay;
- stderr relay;
- abort before launch;
- abort after launch;
- runner/pane loss;
- result mapping;
- no local duplicate after `started=true`.

### 4.5 Worker runner

Use deterministic child fixtures.

Test:

- argv-based `shell:false` spawn;
- cwd and non-Herdr environment preservation;
- parent Herdr variable stripping;
- worker Herdr variable reapplication;
- environment artifact deletion;
- raw stdout written byte-for-byte;
- stderr written byte-for-byte;
- JSONL record parsing across arbitrary chunk boundaries;
- final line without newline;
- malformed JSON line tolerance;
- token-delta suppression;
- lifecycle/tool/retry activity formatting;
- output deduplication and throttling;
- terminal escape stripping;
- secret redaction;
- heartbeat updates;
- atomic state/exit writes;
- SIGINT/SIGTERM/SIGKILL escalation;
- normal and forced exit evidence;
- output backpressure;
- bounded memory use on large streams.

### 4.6 Pane pool

Test:

- one-, two-, three-, and four-slot layouts;
- allocation up to capacity;
- fifth request queues when capacity is four;
- atomic reservation;
- simultaneous acquisition;
- chain reuse;
- retry reuse;
- success expiry/reuse;
- failed/blocked/orphaned slots are not auto-reused;
- pane reset clears prior authority/metadata safely;
- duplicate dispatch request deduplication;
- main focus preservation;
- active pool recovery from durable state.

### 4.7 Installer/CLI

Test:

- compatibility selection;
- exact source fingerprint acceptance/rejection;
- patch check failure;
- staging cleanup;
- side-by-side installation;
- current/previous pointer changes;
- activation rollback on doctor failure;
- conflicting non-owned extension/plugin paths;
- config migration and backup;
- stale installation lock;
- dry-run cleanup;
- unsafe cleanup refusal;
- uninstall preserves official upstream installations and runtime evidence by default.

## 5. Contract tests

Each component consumes test vectors maintained in `packages/protocol`.

Required vectors:

```text
backend-resolve-v1-valid.json
execution-request-v1-valid.json
execution-result-v1-valid.json
launch-v1-valid.json
state-v1-valid.json
heartbeat-v1-valid.json
exit-v1-valid.json
```

Invalid vectors cover:

- missing identity;
- mismatched attempt;
- unsupported version;
- state path outside root;
- invalid final state/evidence combination;
- negative sequence;
- missing exit code;
- unknown enum value where strictness is required.

The GSD patch, extension, runner, plugin, and CLI must all pass the same protocol fixtures.

## 6. Fake fixtures

### 6.1 Fake GSD child

A fixture executable accepts options controlling its behavior:

```text
--scenario success
--scenario tools
--scenario malformed-json
--scenario retryable-error
--scenario stderr-failure
--scenario hang
--scenario ignore-sigint
--scenario ignore-sigterm
--scenario large-output
--scenario final-line-no-newline
```

It emits valid Pi/GSD-compatible JSONL where required and records received signals.

### 6.2 Fake Herdr server/CLI

The fake supports the minimum API surface and fault injection:

```text
normal success
missing capability
slow response
dropped response after side effect
malformed response
pane disappears
pane run fails
state report fails
duplicate response
snapshot mismatch
```

### 6.3 Fake clock

Retention, heartbeat, retry grace, and timeout logic use an injectable clock where practical.

## 7. GSD patch tests

The patch is tested inside an exact upstream GSD-Pi checkout.

### 7.1 No-provider regression

With no external provider installed:

- local execution remains unchanged;
- cmux selection remains unchanged;
- existing subagent tests pass;
- no new error/latency is observable beyond one bounded discovery turn.

### 7.2 Fake provider

Register a fake backend through the event bus and verify:

- external path is selected before cmux/local;
- single, parallel, chain, background, retry, fresh/fork, and isolation paths all use the seam;
- stdout lines feed the existing `processSubagentEventLine()` path;
- stderr, exit code, stop reason, and abort map correctly;
- external metadata is additive;
- `handled=false, started=false` may fall back when allowed;
- `started=true` never falls back.

### 7.3 Result parity

For every deterministic scenario, compare local and external results.

Required parity fields:

```text
final assistant output
messages relevant to result rendering
usage input/output/cache/cost/context/turns
model
thinking
stopReason
errorMessage
exitCode
sessionFile
merge result
run-store terminal status
journal completion counts
```

External-only pane/artifact metadata is excluded from equality but validated separately.

## 8. Herdr contract tests

Against each supported Herdr binary:

1. start a named isolated test session/server;
2. export/read the installed API schema;
3. assert required methods;
4. create workspace/tab/pane fixtures;
5. run a harmless command;
6. read output;
7. send keys;
8. report/release agent/session/metadata;
9. obtain a session snapshot;
10. close only integration-created fixtures.

The test must not inspect or mutate the user's normal Herdr session.

## 9. End-to-end scenarios

### E2E-01 — Root GSD state

- start fake/controlled GSD TUI inside Herdr;
- confirm main pane reports idle;
- start a turn;
- confirm working;
- emit blocked event;
- confirm blocked with label;
- clear block and settle;
- confirm idle;
- shutdown;
- confirm authority release.

### E2E-02 — Single success

- dispatch one worker;
- confirm worker tab/pane creation;
- confirm starting/working state;
- confirm readable activity;
- confirm raw JSON absent from pane;
- confirm raw JSON present in artifact;
- confirm exit 0 and parent result parity;
- confirm retention behavior.

### E2E-03 — Parallel four

- dispatch four deterministic workers;
- confirm four unique slots/panes;
- confirm concurrent activity;
- confirm no metadata/identity collision;
- confirm all parent results.

### E2E-04 — Parallel eight

- dispatch eight with GSD concurrency four;
- confirm only four active panes at once;
- confirm remaining children are queued;
- confirm slot reuse without stale output authority.

### E2E-05 — Chain

- execute a three-step chain;
- confirm one stable pane is reused;
- confirm visible step separators;
- confirm `{previous}` behavior remains owned by GSD;
- confirm final chain output.

### E2E-06 — Retry

- first attempt emits retryable provider failure;
- second succeeds;
- pane remains working/retrying rather than false idle;
- same slot/pane is reused;
- parent records one logical child with correct attempt outcome.

### E2E-07 — Failure

- child exits nonzero with stderr;
- pane shows bounded redacted failure;
- full stderr artifact exists;
- failed pane is retained;
- parent result is failed.

### E2E-08 — Cancellation

- running child receives parent abort;
- runner forwards SIGINT;
- child exits;
- exit artifact records aborted;
- parent resolves without orphan process.

Repeat with fixtures that ignore SIGINT and SIGTERM to validate escalation.

### E2E-09 — Pane closure

- close an active worker pane manually/programmatically;
- parent detects pane loss;
- no local duplicate starts;
- attempt fails with explicit pane-lost evidence.

### E2E-10 — Detach/reattach

- start a long deterministic worker;
- detach client;
- verify process and heartbeat continue;
- reattach;
- verify pane and state remain consistent;
- complete normally.

### E2E-11 — Parent crash

- start worker;
- terminate parent GSD process;
- worker continues under default policy;
- plugin marks it orphaned after timeout/reconciliation;
- evidence remains accessible;
- no false success is published.

### E2E-12 — Herdr restart/reconciliation

- create retained and active fixture records;
- restart controlled Herdr session/server when supported;
- run plugin startup reconciliation;
- verify live/missing/stale associations are classified correctly.

### E2E-13 — Security inputs

Use tasks/tool summaries containing:

```text
quotes
newlines
$(command)
backticks
ANSI/OSC sequences
fake API keys
authorization headers
../../ paths
```

Verify no command injection, terminal injection, secret leakage, or path escape.

## 10. Manual canary plan

Before promoting a new stack:

1. install as canary side-by-side;
2. run `doctor` and automated smoke;
3. use a disposable repository;
4. run one single, one parallel, one failed, and one cancelled dispatch;
5. detach and reattach during a worker;
6. inspect raw and rendered evidence;
7. run for at least one normal GSD milestone or an equivalent bounded workflow;
8. retain the prior stable stack;
9. promote only after no unexplained orphan, stale pane, or result mismatch.

## 11. CI workflows

### Quality

Runs on pull requests and main:

```text
install dependencies
format/lint
typecheck
unit tests
contract fixtures
package build
manifest validation
secret scan
```

### GSD patch compatibility

Matrix:

```text
pinned stable GSD-Pi
a supported later stable when added
current upstream main as informational/canary
```

Steps:

- checkout exact source;
- verify fingerprints;
- apply patch check;
- apply patch;
- run focused subagent tests;
- run typecheck/build boundary needed by patch;
- run fake-backend parity suite.

### Herdr contract

Matrix:

```text
minimum supported Herdr
latest supported stable
current upstream master as informational/canary
```

Steps:

- install/build binary;
- validate API schema;
- run isolated CLI/socket contract tests;
- link plugin and validate manifest/actions.

### macOS E2E

Runs on macOS arm64 where available. Uses only named isolated Herdr state/session locations and deterministic fake children.

### Upstream watch

Scheduled workflow:

- detect new Herdr and GSD-Pi releases;
- test capability/patch compatibility;
- open/update one tracking issue on failure;
- attach concise evidence, not generated dumps;
- never auto-promote a new version.

## 12. Test evidence in planning

A milestone is not complete because code exists. `PLANNING.md` must reference evidence such as:

```text
M3 exit criteria:
- runner unit suite: 84 passed
- raw JSON E2E: passed
- signal escalation E2E: passed
- secret redaction suite: passed
```

Exact command, commit, environment, and known exclusions should be recorded in the progress log or linked report.

## 13. Test anti-patterns

Do not:

- test only mocked success responses;
- copy the production parser into tests;
- assert that a command was called without validating side effects and result parity;
- use the user's live Herdr session for automated tests;
- rely on sleep-only timing when events or fake clocks can provide deterministic evidence;
- treat a successful Ctrl-C send as process termination;
- declare success from exit code alone when GSD expects a valid final response;
- allow fixture secrets that resemble real credentials into repository history;
- skip no-provider regression tests when changing the patch seam.
