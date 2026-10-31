# APM Tracker

## Notes

- Repository setup is complete: local `master` is at `7930b851f` (`chore: stop tracking APM files and add Worker rules`), `origin/master` is still `c4a329c57`, `.apm/` is untracked and ignored, the remote `copilot/modify-inline-conversion-preview` branch is deleted, and `fix/enter-highlighted-commit` is checked out at the setup commit.
- `/tmp/opencode/bazelisk` was missing at initiation. Restored it there as Bazelisk v1.30.0; it starts the cached Bazel 9.0.2. No system package was installed.
- Plan correction: Task 1.1 now adds the `COMMIT_RAW` enumerator to `src/session/keymap.h` so its keymap tests compile. Registration, dispatch and bindings stay in Task 1.2, so behaviour is unchanged in Task 1.1. Source: Task 1.1 must compile while asserting `COMMIT_RAW`.
- Windows UI acceptance remains a User check after Task 2.1; Linux cannot verify it.
- Task 1.1 accepted as Success with accepted limitation: six tests fail exactly as predicted for the later implementation Task (three pass, no unexpected results). Source: .apm/memory/stage-01/task-01-01.log.md (§Important Findings, per-test results). Production unchanged; all work committed at c214d4542 on fix/enter-highlighted-commit. Branch awaits merge into master before Task 1.2 can run.

## Tasks

**Stage 1:**

| Task | Agent | Status | Branch | Worktree | Next Action |
|------|-------|--------|--------|----------|-------------|
| 1.1 | session-agent | Done | fix/enter-highlighted-commit | | Merge fix/enter-highlighted-commit into master, then dispatch 1.2 |
| 1.2 | session-agent | Ready | fix/enter-highlighted-commit | | Dispatch when Worker is free |

## Workers

| Agent | Conversation | Not Read Since Handoff | Context |
|-------|--------------|------------------------|---------|
| session-agent | 1 | | |
