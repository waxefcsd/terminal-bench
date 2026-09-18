# Event Replay Reconciliation — Evaluation Record

Date: 2026-09-18

Task: `tasks/event-replay-reconciliation`

Environment: Harbor 0.23.0. Early validation used Modal; subsequent trials use local Docker to avoid Modal compute charges. Codex authenticates through the subscription `auth.json` path with `CODEX_FORCE_AUTH_JSON=1`, not `OPENAI_API_KEY`. Claude must use a `claude setup-token` OAuth token, not `ANTHROPIC_API_KEY`.

## Required checks

| Check | Result | Evidence |
|---|---:|---|
| Reference-engine unit tests | Pass | 12/12 |
| Cross-implementation oracle comparison | Pass | 2/2; all six hidden case families |
| Verifier Docker build | Pass | `event-replay-reconciliation-tests` |
| TB3 static checks | Pass | 22/22 using LF-normalized temporary copies of the repository CI scripts on Windows |
| Oracle, Modal | 1.0 | `jobs/event-replay-redesign-oracle-modal-7` at `ea8aa5d5` |
| Nop, Modal | 0.0 | `jobs/event-replay-redesign-nop-modal-4` at `ea8aa5d5` |
| Oracle, local Docker | 1.0 | `jobs/event-replay-final-docker-oracle`; 0 exceptions |
| Nop, local Docker | 0.0 | `jobs/event-replay-final-docker-nop`; 0 exceptions |
| Implementation-rubric review | Infrastructure failure | `jobs/2026-09-17__23-45-20`: `AgentSetupTimeoutError` after 360 seconds; no rubric verdict |

The Modal artifact collector logs `SandboxFilesystemNotADirectoryError` when it probes individual artifact paths as directories. This occurs after the agent phase and does not affect verifier execution, exception counts, or rewards.

## Standard agent trials

Command shape:

```powershell
harbor run -p tasks/event-replay-reconciliation `
  --agent codex --model openai/gpt-5.6-sol `
  --env docker --yes `
  --ae CODEX_FORCE_AUTH_JSON=1 `
  --ak reasoning_effort=xhigh
```

| Agent/model | Job | Exceptions | Reward | Classification |
|---|---|---:|---:|---|
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-final-2` | 0 | 0.0 | Historical result; invalidated by later verifier fixes |
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-final-3` | 0 | 0.0 | Historical result; invalidated by later verifier fixes |
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-final-4` | 0 | 0.0 | Historical result; invalidated by later verifier fixes |

After the verifier fixes, two replacement trials were valid model failures on that task revision:

| Agent/model | Job | Exceptions | Reward | Classification |
|---|---|---:|---:|---|
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-fixed-1` | 0 | 0.0 | Valid model failure on the pre-Docker revision |
| Codex / `openai/gpt-5.6-sol`, xhigh | `event-replay-redesign-codex-fixed-2` | 0 | 0.0 | Valid model failure on the pre-Docker revision |

Replacement runs `codex-fixed-3`, `codex-fixed-4`, and `codex-fixed-5` ended with `AgentTimeoutError` and are excluded. A third valid replacement trial is still required.

`codex-fixed-6` ended with `ApiUsageLimitError` and is also excluded. Harbor's `cost_usd` field is an accounting estimate; no model API key was configured for these Codex subscription runs. Local Docker is used going forward so that no additional Modal compute is consumed.

The first local-Docker attempt, `event-replay-redesign-codex-docker-1`, ended during agent setup with `NonZeroAgentExitCodeError` because Debian's HTTP package endpoint disconnected. It is excluded. The final image now uses HTTPS package sources with retries and preinstalls the Codex agent prerequisites. Because this changes the task environment and checksum, all counted trials must be rerun against this final revision.

The first run against the final local-Docker image, `event-replay-final-docker-codex-1`, authenticated and ran for 16 minutes 29 seconds but ended with `AgentTimeoutError`. It is excluded from the required three genuine model failures. Repeating this unchanged configuration is unlikely to produce countable evidence until the timeout behavior is resolved or the task is run in the CI-default Modal environment.

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

1. Run three Codex standard trials and the Codex adversarial trial against the final local-Docker revision. Oracle 1.0 and Nop 0.0 are already reconfirmed on this revision.
2. Configure Claude Code with `claude setup-token` and keep the token out of Git.
3. Run three Claude Opus 5/max standard trials and one adversarial trial.
4. Re-run the implementation-rubric review when Debian package mirrors are reachable and record its verdict.
5. Add the planned mutation suite and broader multi-order/refund/recall/equal-clock fixtures.
6. Publish only after recording the frozen Git SHA/task checksum and confirming no credentials or local job artifacts are included.
