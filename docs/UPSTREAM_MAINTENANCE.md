# Upstream Maintenance

## 1. Objective

`gsd-herdr` must track two fast-moving upstream projects without becoming a permanent divergent fork.

The maintenance model is:

```text
Herdr upstream
  └─ consumed as an external binary/API
      └─ no core patch by default

GSD-Pi upstream
  └─ consumed at an exact version/commit
      └─ one small generic patch queue until upstream exposes an equivalent seam

gsd-herdr
  └─ owns plugins, extensions, runner, installer, compatibility, and tests
```

## 2. Upstream sources

| Project | Repository | Default branch | Initial stable baseline |
|---|---|---|---|
| Herdr | `herdrdev/herdr` | `master` | `v0.8.2` |
| GSD-Pi | `open-gsd/gsd-pi` | `main` | `v1.16.2` |

All compatibility records include both a human-readable version and exact source identity where available.

## 3. Dependency policy

### Stable channel

Stable stack releases use exact, tested versions.

```json
{
  "herdr": {
    "version": "0.8.2",
    "source": "release"
  },
  "gsdPi": {
    "version": "1.16.2",
    "commit": "<exact-commit>",
    "patchSet": "external-subagent-backend-v1"
  }
}
```

No floating `latest`, branch head, or broad untested semver range is activated in production.

### Canary channel

Canary compatibility may test:

- latest upstream stable releases;
- current Herdr `master`;
- current GSD-Pi `main`.

Canary results are informational until explicitly promoted.

## 4. Patch queue policy

### 4.1 Allowed patch scope

The GSD-Pi patch may only contain the generic external-subagent execution seam and tests directly required by that seam.

Allowed concerns:

- backend discovery;
- request/result types;
- structured stdout/stderr callbacks;
- cancellation propagation;
- strict fallback policy;
- additive external-runtime metadata;
- regression tests preserving existing behavior.

Not allowed in the GSD patch:

- Herdr imports;
- pane layout logic;
- Herdr CLI/socket calls;
- activity rendering;
- worker-runner implementation;
- integration configuration parsing beyond a generic fallback signal;
- unrelated GSD cleanup/refactors.

### 4.2 Patch layout

```text
patches/gsd-pi/
├── v1.16.2/
│   ├── manifest.json
│   └── 0001-external-subagent-backend.patch
├── v1.16.3/
│   ├── manifest.json
│   └── 0001-external-subagent-backend.patch
└── main/
    ├── manifest.json
    └── 0001-external-subagent-backend.patch
```

A patch set is copied/ported per supported upstream source rather than ambiguously reused against an unverified range.

### 4.3 Patch manifest

```json
{
  "schemaVersion": 1,
  "id": "external-subagent-backend-v1",
  "bridgeProtocol": 1,
  "upstream": {
    "repository": "open-gsd/gsd-pi",
    "version": "1.16.2",
    "commit": "<exact-commit>"
  },
  "expectedFiles": {
    "src/resources/extensions/subagent/index.ts": "sha256:<digest>",
    "src/resources/extensions/subagent/launch.ts": "sha256:<digest>"
  },
  "patches": [
    "0001-external-subagent-backend.patch"
  ],
  "verification": {
    "commands": [
      "pnpm run typecheck:extensions",
      "<focused subagent test command>"
    ]
  }
}
```

The manifest is evidence, not a suggestion. An unknown source fingerprint stops the automated patch process.

### 4.4 Application rules

Production patch application requires:

1. exact repository/source identity;
2. matching expected fingerprints;
3. clean `git apply --check` or exact equivalent;
4. no rejected hunks;
5. focused tests;
6. successful build/overlay validation;
7. successful stack smoke test.

Production automation must not use fuzzy patch application as a fallback.

### 4.5 Patch generation

Preferred workflow:

```bash
git clone https://github.com/open-gsd/gsd-pi.git /tmp/gsd-pi
cd /tmp/gsd-pi
git checkout <exact-ref>
git switch -c gsd-herdr/external-backend
# apply focused source changes
git commit -m "feat(subagent): expose external execution backend"
git format-patch -1 --stdout > /path/to/0001-external-subagent-backend.patch
```

The patch commit should be independently comprehensible and free of generated dependency changes unless required.

## 5. Herdr compatibility policy

Herdr is validated primarily by capabilities, secondarily by version.

### Required methods

The compatibility manifest names required API methods. The doctor and CI inspect the installed schema and verify that equivalent methods exist.

Examples:

```text
session.snapshot
tab.create
tab.list
pane.split or layout.apply
pane.run
pane.read
pane.send_keys
pane.process_info
pane.report_agent
pane.report_agent_session
pane.report_metadata
pane.release_agent
pane.close
```

### Capability changes

A new Herdr release is compatible only when:

- required methods remain available;
- request/response shapes used by the client remain valid;
- pane ID/context behavior remains compatible;
- plugin manifest fields remain supported;
- status reporting and release behavior pass contract tests;
- isolated E2E tests pass.

A version number alone is insufficient.

### Herdr core patches

`patches/herdr/` remains empty by default.

A Herdr patch is permitted only when:

1. a required behavior cannot be implemented through documented plugin/CLI/socket surfaces;
2. the limitation is reproduced and documented;
3. a minimal patch can be maintained independently;
4. the plan and decision record are updated;
5. compatibility/rollback implications are defined.

Before adding such a patch, prefer:

- adjusting the plugin;
- using another existing public method;
- filing a concise Discussion/feature proposal upstream;
- delaying the feature rather than expanding core divergence.

## 6. Scheduled upstream watch

A scheduled workflow should run daily or weekly.

### Herdr watch

1. discover newest stable release and current `master` commit;
2. obtain/build the binary in isolation;
3. export API schema;
4. run capability and plugin contract tests;
5. run selected E2E fixtures;
6. store a compatibility report.

### GSD-Pi watch

1. discover newest stable release and current `main` commit;
2. fetch exact source;
3. compare target-file fingerprints;
4. attempt patch check in isolation;
5. compile/typecheck the patched surface;
6. run fake-backend and result-parity tests;
7. store a compatibility report.

### Failure reporting

The workflow maintains one tracking issue per upstream compatibility line rather than opening duplicates.

A useful report contains:

```text
upstream project/ref
first failing step
smallest relevant error
whether current stable remains supported
whether failure is API, patch, test, or infrastructure
link to CI evidence
```

It should not automatically modify the active support matrix.

## 7. Adopting a new Herdr release

Process:

1. scheduled/manual contract test passes;
2. review Herdr release notes for terminal, API, plugin, pane, and persistence changes;
3. add a candidate compatibility entry;
4. create a canary stack build;
5. run automated E2E on macOS arm64;
6. run manual canary through representative GSD workflows;
7. update docs for behavior/config changes;
8. promote in a new `gsd-herdr` release.

Because Herdr remains unpatched, adoption should usually require compatibility metadata and testing rather than code changes.

## 8. Adopting a new GSD-Pi release

Process:

1. freeze exact upstream ref;
2. inspect changes to:
   - bundled subagent implementation;
   - launch contract;
   - run store;
   - retry/parallel/chain paths;
   - isolation/merge behavior;
   - extension event bus;
   - packaging/runtime resource loading;
3. port the seam as a fresh focused commit;
4. generate a version-specific patch;
5. update fingerprints and manifest;
6. run upstream relevant tests unmodified where possible;
7. run no-provider regression tests;
8. run fake-backend tests;
9. run local-vs-external result parity;
10. create and manually exercise a canary;
11. add the new compatibility entry;
12. release without removing the prior supported stack immediately.

Do not mechanically resolve conflicts without understanding semantic changes in every supported subagent mode.

## 9. When upstream adds a native seam

The desired end state is no GSD-Pi patch.

Migration criteria:

- upstream hook supports every required mode;
- launch plan preserves model/thinking/session/tool behavior;
- stdout/stderr/result callbacks are sufficient;
- cancellation is supported;
- strict fallback policy is representable;
- result parity suite passes without the patch;
- extension can register without load-order hacks.

Migration steps:

1. add adapter support for the upstream API;
2. support both patched-v1 and native API during one transition release if needed;
3. run full parity/E2E matrix;
4. mark patch set deprecated;
5. remove patch requirement from newer compatibility entries;
6. keep old stack versions immutable for rollback;
7. delete only obsolete patch source after support policy permits.

## 10. Branching strategy for this repository

Recommended:

```text
main
  releasable integration code and docs

feature/*
  normal development branches

compat/gsd-<version>
  temporary patch-port work

compat/herdr-<version>
  temporary compatibility adaptation work

release/*
  release preparation when needed
```

Do not mirror complete upstream branches into this repository.

Temporary upstream checkouts belong in CI workspaces or local build caches, not git history.

## 11. Release compatibility records

Every `gsd-herdr` release includes an immutable `stack.lock.json`.

Example:

```json
{
  "stackVersion": "0.1.0",
  "bridgeProtocol": 1,
  "configSchema": 1,
  "artifactSchema": 1,
  "platform": "darwin-arm64",
  "node": ">=22.18.0",
  "herdr": {
    "version": "0.8.2",
    "requiredCapabilitySet": "herdr-pane-runtime-v1"
  },
  "gsdPi": {
    "version": "1.16.2",
    "commit": "<exact-commit>",
    "patchSet": "external-subagent-backend-v1"
  },
  "packages": {
    "protocol": "0.1.0",
    "gsdExtension": "0.1.0",
    "workerRunner": "0.1.0",
    "herdrPlugin": "0.1.0",
    "cli": "0.1.0"
  }
}
```

## 12. Support policy proposal

Before `1.0`:

- support one primary stable stack;
- retain at least one prior known-good stack for rollback;
- test current upstream heads as best-effort canary;
- do not promise broad version ranges.

After `1.0`:

- define a rolling support window based on tested combinations;
- publish exact compatibility tables;
- deprecate stacks with notice;
- retain installers/locks/checksums for supported rollback versions.

## 13. Documentation updates required during upstream adoption

Update:

- `PLANNING.md` progress, risks, and compatibility baseline;
- `README.md` supported versions;
- this file if process changes;
- `INTEGRATION_CONTRACT.md` for protocol changes;
- `CONFIGURATION.md` for new options/migrations;
- release notes with test evidence and known limits.

## 14. Maintenance anti-patterns

Do not:

- keep full upstream source in this repository;
- maintain permanent `penggin/herdr` and `penggin/gsd-pi` runtime forks as the default distribution;
- patch installed package files in place;
- apply one patch against an unbounded version range;
- auto-promote because a patch applies cleanly;
- treat Herdr version compatibility as proof of API compatibility;
- combine GSD seam ports with unrelated upstream fixes;
- force-push or rewrite released patch sets;
- delete the previous known-good stack before canary promotion;
- continue silently when protocol or fingerprint checks fail.
