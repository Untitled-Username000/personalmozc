APM_RULES {

## Task scope
- Work only on the Task described in your Task Prompt. Do not read `.apm/spec.md`, `.apm/plan.md`, or other Tasks; your Task Prompt carries the context you need.
- Stay inside the files, repository and commands named in your Task Prompt. Report a needed change outside that scope instead of making it.
- Treat the Task Prompt as authoritative over any conflicting repository convention. If they conflict, stop and report before changing code.

## Workspace and Git
- Work in `/workspaces/personalmozc`, the single Git repository for this project.
- The base branch is `master`. Work on the branch named in your Task Prompt, for example `fix/enter-highlighted-commit`. If your Task Prompt gives a worktree, use that directory for all Git and build commands.
- Never push. Only the final release Task may push, and only when its Task Prompt says so. Never force-push, rewrite history, delete branches or tags, or run `git reset --hard` on a branch you did not create.
- Do not stage `.apm/` or any planning file. It is untracked and listed in `.gitignore`.
- Stage only files your Task changes. Commit with `type: description`, using `feat`, `fix`, `refactor`, `docs`, `test` or `chore`; keep one logical change per commit.
- Do not merge into `master` yourself unless the Task Prompt explicitly assigns that step.

## Build and tests
- Run commands from `/workspaces/personalmozc/src`.
- Use `/tmp/opencode/bazelisk` as the Bazel launcher. Do not install a different Bazel, modify `.bazeliskrc`, or change `MODULE.bazel` dependencies.
- Run the exact test targets named in your Task Prompt. For long commands, run them in the background and record the log path.
- The first build is slow; wait for it rather than assuming failure. Do not add remote-cache or config overrides unless the Task Prompt names them.
- Report the exact command and its result. Do not treat a compile-only result as passing tests when the Task Prompt asks for tests.
- If a test looks flaky, rerun it once and report both results.
- Run `clang-format` on changed C++ files. The repository `.clang-format` is authoritative.
- Do not install system packages or change the environment without saying so in your report.

## Code changes
- Follow the existing Mozc structure, naming and comment style, and the Google C++ Style Guide referenced by `CONTRIBUTING.md`.
- Make the smallest change that satisfies the Task. Do not reformat unrelated lines, rename unrelated symbols, or clean up nearby code.
- Do not change mobile keymaps, number shortcuts, Tab/Space navigation, candidate rendering or unrelated behaviour unless the Task Prompt explicitly includes it.
- Keep `.tsv` rows consistent with the ordering used by the surrounding section.
- Remove temporary debug output and dead code before committing. Do not leave commented-out code.
- Update comments that describe behaviour you change; do not leave comments that contradict the code.

## Validation and User actions
- Before reporting success, run every check in the Task Prompt and state the observed result.
- Separate checks you ran from checks that need the User or CI. Pause and say exactly what the User must do when the Task Prompt marks an action as the User's.
- Do not claim Windows UI behaviour was verified from Linux. Report the Linux result and name the Windows check as pending.
- If a check cannot run, report why, what you tried, and the remaining risk.

## Reporting
- Write your Task Report in `.apm/bus/<your-agent-slug>/report.md` using the Task Report format your Task Prompt specifies. Keep it factual: files changed, commands run, results, open issues and the commit SHA.
- Clear the incoming `task.md` only after you have processed it, and write the report only after you have verified its content.
- Record test failures with the test name, the expected and actual values, and the relevant source path.
- Do not report a Task complete when a required test fails. Report the failure and the blocker instead.

## Safety
- Do not touch credentials, signing keys, release assets or the `upstream` remote configuration.
- Do not delete files, branches, worktrees or Git objects unless the Task Prompt names them.
- Never commit generated build output, Bazel caches or logs.

} //APM_RULES
