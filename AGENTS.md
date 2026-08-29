# Agent Instructions

These instructions apply to every task in this repository.

## Start-of-session protocol

Before editing any file:

1. Read `PLANNING.md` completely.
2. Identify the current milestone, exact task IDs, dependencies, and exit criteria.
3. Read the documentation linked by the affected task.
4. Inspect the current repository state rather than relying on an earlier conversation summary.
5. When work depends on Herdr or GSD-Pi internals, verify the current pinned/upstream source before making factual claims or code changes.

Do not start implementation work that belongs to a later milestone unless `PLANNING.md` explicitly records the prerequisite decision and marks the task ready.

## Source-of-truth hierarchy

1. `PLANNING.md` — current status, task order, risks, and next action.
2. `docs/DECISIONS.md` — accepted architecture constraints.
3. `docs/INTEGRATION_CONTRACT.md` — cross-component protocol behavior.
4. Other design documents under `docs/`.
5. Code and tests once implementation begins.

When sources conflict, stop and update the plan/decision record before proceeding.

## Change discipline

- Make the smallest coherent change that completes one or more listed task IDs.
- Preserve the plugin-first, minimal-patch architecture.
- Do not vendor complete Herdr or GSD-Pi source trees.
- Do not add Herdr-specific code to the GSD-Pi patch seam.
- Do not copy or replace GSD-Pi's bundled `subagent` tool.
- Do not introduce invisible local fallback when monitoring is required.
- Do not display raw JSONL or token deltas in worker panes.
- Do not weaken security, path-containment, redaction, or exact-patch requirements for convenience.
- Treat process spawning, signal forwarding, artifact cleanup, environment transfer, and patch application as security-sensitive work.

## Testing protocol

Before marking a task complete:

1. Run the focused tests for the changed component.
2. Run contract tests for any changed serialized/API boundary.
3. Run no-provider regression and result-parity tests when changing the GSD seam.
4. Run security tests when changing process, environment, path, signal, redaction, installation, or cleanup behavior.
5. Record the exact test command and result in the plan progress log or a linked report.

Do not claim that a milestone is complete without evidence for every exit criterion.

## End-of-session protocol

Before ending a development session, update `PLANNING.md`:

- check completed task IDs;
- leave incomplete tasks unchecked;
- add newly discovered tasks and dependencies;
- update risks and mitigations;
- update decisions or add an ADR when architecture changed;
- add a dated progress-log entry;
- record tests/evidence;
- state the exact next task;
- update the current milestone/status if appropriate.

Then review the diff to ensure the plan accurately describes the repository.

## Current phase restriction

The repository is currently in **M0 — Repository foundation and feasibility validation**.

Until M0.6–M0.9 are completed and recorded:

- documentation and technical-spike work are allowed;
- implementation package scaffolding should wait;
- the GSD-Pi patch distribution mechanism must not be assumed;
- Herdr capabilities must be verified from the actual schema;
- no production installation claim may be made.
