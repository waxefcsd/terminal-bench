# Event Replay Reconciliation — Evaluation Record

Date: 2026-09-17

Task: `tasks/event-replay-reconciliation`

Environment: Harbor 0.23.0, Modal, Codex subscription authentication via `CODEX_FORCE_AUTH_JSON=1`.

## Required checks

| Check | Result | Evidence |
|---|---:|---|
| Reference-engine unit tests | Pass | 12/12 |
| Independent oracle comparison | Pass | 2/2; all six hidden case families |
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
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-final-2` | 0 | 0.0 | Valid model failure |
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-final-3` | 0 | 0.0 | Valid model failure |
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-final-4` | 0 | 0.0 | Valid model failure |

An earlier run, `event-replay-redesign-codex-final-1`, ended with `ApiUsageLimitError`; it is explicitly excluded from the three required trials.

All three valid Codex runs generated syntactically valid canonical JSON but failed exact semantic checks across the hidden happy-path, compensation, precedence, migration, validation, and identity cases. Two also omitted required README terminology. The consistent failure mode is incomplete agreement on the full audit/manifest contract and order-independent replay semantics, not an agent crash or infrastructure error.

Claude Code trials are not yet run. This machine currently has neither the `claude` command nor `CLAUDE_CODE_OAUTH_TOKEN`; a personal OAuth token must be supplied outside the repository before the required three trials can run.

## Adversarial trials

The Codex adversarial run appended the repository's current `docs/prompts/hack-trial-prompt.md`, matching the current `/cheat` workflow's red-team brief.

| Agent/model | Job | Reward | Note |
|---|---|---:|---|
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-cheat-1` | 0.0 | Harbor classified the exit as `AgentSafetyRefusalError`; no exploit artifact was produced |

The numerical zero-reward condition is met for Codex, but the refusal classification is retained here for transparency. The Claude adversarial run remains blocked on Claude OAuth authentication.

## Remaining work before submission

1. Configure Claude Code with `claude setup-token` and keep the token out of Git.
2. Run three Claude Opus 5/max standard trials and confirm each has zero exceptions and reward 0.
3. Run one Claude Opus 5/max adversarial trial and confirm reward 0.
4. Re-run the implementation-rubric review when Debian package mirrors are reachable and record its verdict.
5. Publish the repository only after reviewing `git status` to ensure no credentials or local job artifacts are included.
