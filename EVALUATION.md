# Event Replay Reconciliation — Evaluation Record

Date: 2026-09-17

Task: `tasks/event-replay-reconciliation`

Environment: Harbor 0.23.0, Modal, Codex subscription authentication via `CODEX_FORCE_AUTH_JSON=1`.

## Required checks

| Check | Result | Evidence |
|---|---:|---|
| Reference-engine unit tests | Pass | 12/12 |
| Cross-implementation oracle comparison | Pass | 2/2; all six hidden case families |
| Verifier Docker build | Pass | `event-replay-reconciliation-tests` |
| TB3 static checks | Pass | 22/22 using LF-normalized temporary copies of the repository CI scripts on Windows |
| Oracle, Modal | 1.0 | `jobs/event-replay-redesign-oracle-modal-4` |
| Nop, Modal | 0.0 | `jobs/event-replay-redesign-nop-modal-2` |
| Implementation-rubric review | Infrastructure failure | Reviewer image failed while fetching Debian packages; no rubric verdict was produced |

The Modal artifact collector logs `SandboxFilesystemNotADirectoryError` when it probes individual artifact paths as directories. This occurs after the agent phase and does not affect verifier execution, exception counts, or rewards.

## Standard agent trials

Command shape:

```powershell
harbor run -p tasks/event-replay-reconciliation `
  --agent codex --model openai/gpt-5.6-sol `
  --env modal --yes `
  --ae CODEX_FORCE_AUTH_JSON=1 `
  --ak reasoning_effort=xhigh
```

| Agent/model | Job | Exceptions | Reward | Classification |
|---|---|---:|---:|---|
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-final-2` | 0 | 0.0 | Historical result; invalidated by later verifier fixes |
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-final-3` | 0 | 0.0 | Historical result; invalidated by later verifier fixes |
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-final-4` | 0 | 0.0 | Historical result; invalidated by later verifier fixes |

An earlier run, `event-replay-redesign-codex-final-1`, ended with `ApiUsageLimitError`; it is explicitly excluded from the three required trials.

These three runs completed without infrastructure exceptions, but later review found verifier defects and caused semantic changes. They are retained only as historical evidence and must not be counted toward the final three-run requirement.

Claude Code trials are not yet run. This machine currently has neither the `claude` command nor `CLAUDE_CODE_OAUTH_TOKEN`; a personal OAuth token must be supplied outside the repository before the required three trials can run.

## Adversarial trials

The Codex adversarial run appended the repository's current `docs/prompts/hack-trial-prompt.md`, matching the current `/cheat` workflow's red-team brief.

| Agent/model | Job | Reward | Note |
|---|---|---:|---|
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-cheat-1` | 0.0 | Harbor classified the exit as `AgentSafetyRefusalError`; no exploit artifact was produced |

The numerical zero-reward condition is met for Codex, but the refusal classification is retained here for transparency. The Claude adversarial run remains blocked on Claude OAuth authentication.

## Remaining work before submission

1. Re-run Oracle, Nop, three Codex standard trials, and the Codex adversarial trial after the verifier fixes.
2. Configure Claude Code with `claude setup-token` and keep the token out of Git.
3. Run three Claude Opus 5/max standard trials and one adversarial trial.
4. Re-run the implementation-rubric review when Debian package mirrors are reachable and record its verdict.
5. Add the planned mutation suite and broader multi-order/refund/recall/equal-clock fixtures.
6. Publish only after recording the frozen Git SHA/task checksum and confirming no credentials or local job artifacts are included.
