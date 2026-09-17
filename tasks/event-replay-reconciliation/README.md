# Event Replay Reconciliation

## Difficulty explanation

The task combines lossless schema migration, global identity reconciliation, causal scheduling, four related state machines, cross-aggregate barriers, and one-use compensation semantics. Correctness depends on the historical replay point rather than only the final collection of events.

## Solution explanation

The reference solution normalizes legacy events, isolates conflicting identities, then advances a deterministic causal ledger one terminal decision at a time. It separately classifies failed dependencies and causal cycles before emitting canonical state, audit, and integrity artifacts.

## Verification explanation

The separate verifier uses an independently structured reference engine and compares every output byte across hidden lifecycle, migration, compensation, precedence, and metamorphic cases. Submitted code runs without root privileges and cannot access verifier data or the reward channel.

## Relevant experience

This task was designed from practical data-engineering patterns: event-version migrations, idempotent delivery, compensating transactions, and recovery audit trails. The author implemented and evaluated the complete task using reproducible containerized checks.
