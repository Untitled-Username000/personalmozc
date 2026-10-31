# APM Tracker

## Notes

- Repository setup is complete: local `master` is at `7930b851f` (`chore: stop tracking APM files and add Worker rules`), `origin/master` is still `c4a329c57`, `.apm/` is untracked and ignored, the remote `copilot/modify-inline-conversion-preview` branch is deleted, and `fix/enter-highlighted-commit` is checked out at the setup commit.
- `/tmp/opencode/bazelisk` was missing at initiation. Restored it there as Bazelisk v1.30.0; it starts the cached Bazel 9.0.2. No system package was installed.
- Plan correction: Task 1.1 now adds the `COMMIT_RAW` enumerator to `src/session/keymap.h` so its keymap tests compile. Registration, dispatch and bindings stay in Task 1.2, so behaviour is unchanged in Task 1.1. Source: Task 1.1 must compile while asserting `COMMIT_RAW`.
- Windows UI acceptance remains a User check after Task 2.1; Linux cannot verify it.

## Tasks

**Stage 1:**

| Task | Agent | Status | Branch | Worktree | Next Action |
|------|-------|--------|--------|----------|-------------|
| 1.1 | session-agent | Active | fix/enter-highlighted-commit | | Execute; await report |
| 1.2 | session-agent | Waiting: 1.1 | fix/enter-highlighted-commit | | Dispatch after 1.1 review |

## Workers

| Agent | Conversation | Not Read Since Handoff | Context |
|-------|--------------|------------------------|---------|
| session-agent | 1 | | |
