# Event Replay Reconciliation Redesign

## Objective

Redesign `tasks/event-replay-reconciliation` into a realistic, specification-complete Terminal-Bench 3 task whose difficulty comes from versioned event migration, cross-aggregate causality, compensation semantics, and deterministic reconciliation. The task must remain solvable by a focused domain expert, must not rely on hidden rules or infrastructure failures, and must satisfy the current repository CI, standard-trial, and adversarial-trial requirements on one frozen task checksum.

## Scope

The task models recovery of a corrupted event export for a multi-aggregate commerce system. It contains four aggregate types—order, payment, inventory, and shipment—belonging to tenant-scoped orders. The submitted program must migrate old events into a canonical representation, isolate conflicting identities, replay a causal graph, enforce cross-aggregate barriers, validate compensation chains, and emit byte-stable state, audit, and manifest artifacts.

The redesign deliberately excludes external services, large-data performance constraints, network dependencies, and arbitrary hidden behavior. Difficulty must come from combining fully documented semantics correctly.

## Task Interface

The agent must implement `/app/replay.py`. The verifier invokes it as:

```sh
python3 /app/replay.py --input-dir <input_dir> --output-dir <output_dir>
```

The default input directory is `/app/input`; the default output directory is `/app`. The input directory contains `schema.json` and `events.jsonl`. A successful invocation writes:

- `/app/state.json`
- `/app/audit.json`
- `/app/manifest.json`
- `/app/README.md`

The program must use only resources installed in the task environment, must not require the network, and must not read `/tests` or `/solution`. It must not modify either input file.

## Canonical Event Model

After migration, every event has a canonical envelope containing:

- `event_id`: globally unique string identity;
- `tenant_id`: tenant namespace;
- `aggregate_type`: one of `order`, `payment`, `inventory`, or `shipment`;
- `aggregate_id`: identity within the aggregate type;
- `order_id`: owning order;
- `event_type`: canonical event type;
- `schema_version`: canonical schema version;
- `logical_clock`: non-negative integer causal clock;
- `occurred_at`: normalized timestamp string;
- `predecessor_ids`: explicit causal dependencies;
- `compensates_event_id`: nullable compensation target;
- `payload`: event-specific canonical payload.

The final instruction must define whether unknown extra fields are rejected or ignored. The selected rule is strict rejection so that event identity, migration, and output are unambiguous.

## Aggregate State Machines

### Order

The order state machine is `draft -> confirmed -> fulfilled`, with a documented transition to `cancelled` when the required compensation state has been reached.

### Payment

The payment state machine is `none -> authorized -> captured`. An uncaptured authorization can become `voided`; a captured payment can become `refunded`.

### Inventory

The inventory state machine is `available -> reserved -> committed`. An uncommitted reservation can become `released`.

### Shipment

The shipment state machine is `none -> planned -> dispatched -> delivered`. A dispatched but undelivered shipment can become `recalled`.

Each payment, inventory, and shipment aggregate belongs to exactly one tenant-scoped order. References across tenants or orders are invalid even when the referenced event ID exists.

## Cross-Aggregate Barriers

- `order.confirmed` requires an applied payment authorization and inventory reservation for the same tenant-scoped order.
- `shipment.planned` requires the order to be confirmed.
- `shipment.dispatched` requires the related inventory to be committed.
- `order.fulfilled` requires payment capture, inventory commit, and shipment delivery.
- A barrier is evaluated against facts that have already been applied at that point in deterministic replay. A future event cannot retroactively satisfy an earlier candidate.

Explicit `predecessor_ids`, schema history requirements, and cross-aggregate barriers are separate gates. Satisfying one gate does not imply either of the others.

## Version Migration

Events may use three public schema versions:

- Version 1 represents money with `amount` as a decimal string plus `currency`.
- Version 2 represents money with integer `amount_cents` plus `currency`.
- Version 3 represents money as `money: {"minor": <integer>, "currency": <string>}` and includes `tenant_id` directly.

`schema.json` declares version aliases, field migrations, and tenant defaults needed to produce the canonical version. The instruction will define the supported migration operations and decimal conversion rules completely. Migration must be lossless: malformed decimals, unsupported precision, absent tenant mappings, conflicting legacy and canonical fields, or unsafe numeric conversions produce `migration_failed`.

Identity grouping occurs after successful migration. Copies with the same `event_id` and identical canonical content represent duplicate delivery. Copies with the same `event_id` but different canonical content represent an identity conflict, and no variant may affect trusted state.

## Compensation Semantics

Compensation events contain `compensates_event_id` and follow these rules:

- `payment.voided` compensates an applied, uncaptured `payment.authorized` event.
- `payment.refunded` compensates an applied `payment.captured` event.
- `inventory.released` compensates an applied, uncommitted `inventory.reserved` event.
- `shipment.recalled` compensates an applied, undelivered `shipment.dispatched` event.
- A compensation target must belong to the same tenant-scoped order.
- A target can be consumed by at most one successfully applied compensation.
- A rejected event cannot be a valid compensation target.
- Legality is evaluated at the compensation event's replay point, not from the final state.
- Order cancellation is valid only when the compensations required by the order's applied side effects have themselves applied.

The final schema and instruction must enumerate the exact required compensation set for cancellation before authorization, after authorization, after capture, before dispatch, and after dispatch. Delivery is terminal and cannot be cancelled.

## Deterministic Replay

Processing has six conceptual phases:

1. Parse and validate basic envelopes without depending on input line order.
2. Migrate supported legacy events into the canonical representation.
3. Group canonical identities, collapse identical copies, and isolate conflicts.
4. Validate tenant/order ownership and construct explicit causal dependencies.
5. Repeatedly select ready events, enforce history requirements, barriers, state transitions, payload consistency, and compensation legality, then apply one event.
6. Classify all events that cannot be applied after progress stops.

A candidate is ready only when all explicit predecessors have applied successfully. Ready candidates are sorted by `(logical_clock, occurred_at, event_id)`. Exactly one candidate is evaluated at a time, and readiness is recomputed after every successful application or dynamic rejection.

## Unique Disposition Rules

Every unique identity receives exactly one terminal disposition. Rejected events use the following precedence:

1. `identity_conflict`
2. `invalid_envelope`
3. `migration_failed`
4. `unknown_event_type`
5. `invalid_reference`
6. `payload_conflict`
7. `invalid_compensation`
8. `invalid_transition`
9. `blocked_dependency`
10. `causal_cycle`
11. `barrier_unsatisfied`

Static errors are assigned before replay. Payload, compensation, and state-transition errors are evaluated only after explicit predecessors have applied. When replay can no longer progress, failure propagation marks missing or rejected predecessors as `blocked_dependency`; strongly connected components in the remaining explicit dependency graph are marked `causal_cycle`; remaining acyclic events with unmet history or cross-aggregate gates are marked `barrier_unsatisfied`.

Applied identities receive `applied`. A collapsed duplicate identity receives the event's terminal disposition plus one deterministic duplicate record containing `copy_count`; duplicate copies do not create repeated audit entries. Conflicting variants record sorted SHA-256 digests of all canonical variants without using source line numbers.

## Output Contracts

### `state.json`

State is ordered by tenant ID and order ID. Each order contains the final state of its order, payment, inventory, and shipment aggregates; applied event IDs; active business facts; and successful compensation relationships. Rejected events contribute no trusted state.

### `audit.json`

Audit contains one terminal record per unique event identity and a deterministic duplicate-delivery record where applicable. Each record includes the canonical identity, aggregate ownership, terminal code, canonical version, and code-specific structured detail. The instruction and schema define every field and sorting key; optional free-form messages are not part of correctness.

### `manifest.json`

The manifest contains:

- a SHA-256 digest of the canonical input multiset rather than the raw JSONL bytes;
- original row count and unique identity count;
- migrated, duplicate, and conflict counts;
- counts for every terminal disposition;
- the complete global application order;
- logical-content digests for state and audit.

The manifest cannot recursively contain its own digest.

### Serialization

All artifacts use UTF-8 JSON, lexicographically sorted object keys, two-space indentation, no insignificant trailing spaces, and exactly one trailing newline. All explicitly unordered arrays have documented sorting keys. Input reordering, JSON object-key reordering, insignificant input whitespace, repeated execution, and injection of extra identical copies must preserve the documented byte-level invariants.

## Verifier Architecture

The verifier runs in a separate container and receives only artifacts declared in `task.toml`. Its reference engine is implemented independently from the oracle solution and is baked into `tests/Dockerfile`; it does not import or execute `/solution` code.

For every fixture, the verifier runs `/app/replay.py` against a fresh temporary input and output directory. It compares all three submitted JSON files with the complete reference result and compares their exact bytes with canonical reference serialization.

Hidden fixture families cover:

- convergence of all supported source schema versions;
- multi-level cross-aggregate barriers;
- legal and illegal compensation chains at each lifecycle stage;
- cross-version duplicates and post-migration identity conflicts;
- missing predecessors, rejection propagation, and causal cycles;
- deterministic concurrency with equal logical clocks;
- tenant and order boundary violations;
- amount, currency, and inventory-quantity conflicts;
- error-precedence cases in which multiple superficial diagnoses are possible.

Each base fixture has deterministic metamorphic variants: full reversal, fixed-seed permutations, injection of one to three identical copies, alternate legal JSON whitespace/key order, and execution in distinct directories. Metamorphic expectations are computed from documented invariants, not relaxed subset assertions.

The input-immutability test hashes the exact temporary `schema.json` and `events.jsonl` given to the submission before and after execution. README verification requires the complete CLI command, every default path, all three output paths, and explanations of migration, idempotency, compensation, and deterministic scheduling.

## Verifier Mutation Tests

Before model trials, the verifier must be run against deliberately faulty implementations that each violate one rule:

- treats schema history requirements as direct predecessors;
- evaluates barriers from final state instead of replay-time state;
- groups identities before migration;
- trusts one variant of a conflicting identity;
- allows a compensation target to be consumed twice;
- labels dependency cycles as missing predecessors;
- compares only parsed JSON and emits unstable bytes;
- mutates the input copy supplied by the verifier;
- hard-codes the public fixture;
- omits applied-event audit records or manifest counts.

Every mutant must receive zero reward. These mutants are local verifier-development assets and are not shipped in the agent environment.

## Difficulty Calibration

Difficulty comes from integrating documented semantics rather than obscurity, excessive data, external services, or unavailable dependencies. The public fixture demonstrates only a simple successful multi-aggregate chain. Hidden fixtures combine public rules but never introduce a new rule.

All formal results must use one frozen task checksum. Results from earlier checksums are diagnostic only and cannot count toward final requirements.

After static checks, rubric review, Docker validation, Oracle `1.0`, and Nop `0.0`, one Codex and one Claude trial are used for calibration. A nonzero reward is analyzed as one of: correct solution, specification defect, verifier defect, or exploit. Specification and verifier defects are fixed directly. A correct model solution may justify adding another realistic semantic interaction, but never an undisclosed rule. Any task change invalidates all previous formal trial results.

The final frozen version must record:

- three completed Codex trials using `openai/gpt-5.6-sol` with `reasoning_effort=xhigh`, all with reward `0.0`;
- three completed Claude Code trials using `anthropic/claude-opus-5` with `reasoning_effort=max` and the current output-token environment default, all with reward `0.0`;
- one completed adversarial trial for each configuration, both with reward `0.0`;
- no authentication, rate-limit, agent crash, container, timeout, or infrastructure error counted as a model failure.

The current `.github/harbor-run-defaults.yml` remains the source of truth for agent, model, reasoning, trial count, analysis, and Modal execution settings.

## Validation and Evidence

Validation proceeds in this order:

1. Run all `scripts/checks/check-*.sh` checks against the task.
2. Run the implementation rubric using the repository rubric file.
3. Build both task and verifier images.
4. Run Oracle and require reward `1.0`.
5. Run Nop and require reward `0.0`.
6. Run verifier mutation tests.
7. Run one Codex and one Claude calibration trial.
8. Freeze the final task checksum after resolving root causes.
9. Run the formal three-by-two standard trial matrix.
10. Run one adversarial trial per agent configuration.
11. Run `harbor analyze` and manually review every trajectory.
12. Document commands, versions, checksums, rewards, exceptions, and failure analysis.

The repository contains:

- `results/final-validation.md` for static, rubric, build, Oracle, Nop, and mutation-test evidence;
- `results/standard-trials.md` for the six standard trials;
- `results/adversarial-trials.md` for the two adversarial trials;
- `results/failure-analysis.md` for trajectory-level analysis;
- `results/reproducibility.md` for non-secret commands, versions, and environment details.

No API key, OAuth token, `.modal.toml`, Codex `auth.json`, raw authorization header, or copied credential may appear in Git, Markdown, job artifacts intended for publication, or result files.

## Submission Hygiene

Before publication, `task.toml` must identify the actual repository owner or task author and must not claim OpenAI authorship unless OpenAI is genuinely the submitting author. The task README's relevant-experience section must be written by the submitter and accurately describe their experience. The final repository must be controlled by the applicant, include an explicit license choice, and expose a clean reproduction path without requiring committed secrets.

## Acceptance Criteria

The redesign is accepted only when:

- the instruction completely specifies every tested behavior and error precedence;
- the independent verifier rejects every listed mutant;
- the oracle and verifier agree on every base and metamorphic fixture;
- static checks, rubric review, image builds, Oracle, and Nop satisfy current CI;
- all six final standard trials are genuine verifier failures with zero reward;
- both final adversarial trials complete with zero reward;
- failure analysis demonstrates meaningful reasoning failures rather than ambiguity or infrastructure problems;
- published documentation matches the final checksum and recorded job outputs.
