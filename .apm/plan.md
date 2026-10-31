# APM Plan

Fix the desktop inline prediction menu so Enter commits the highlighted candidate and Shift+Enter commits the raw composition, then release and verify a Windows x64 build.

## Workers

| Agent | Domain | Description |
|-------|--------|-------------|
| session-agent | Mozc session layer, keymaps and unit tests | Encodes the key contract as tests, diagnoses the current raw-commit fallback, adds the raw-commit command, updates the four desktop keymaps and makes the tests pass. |
| release-agent | Verification, Git integration, CI and release | Independently re-runs the test targets, reviews the diff, merges the branch into `master`, pushes once, monitors Linux CI and the Windows x64 release, and reports the Windows acceptance checklist. |

## Stages

| Stage | Name | Tasks | Agents |
|-------|------|-------|--------|
| 1 | Lock contract and implement Enter/Shift+Enter split | 1.1–1.2 | session-agent |
| 2 | Verify, release and acceptance | 2.1 | release-agent |

## Notes

- Run the two Stage 1 Tasks in order with one Worker. There is no useful parallelism: the implementation depends on the diagnosis, and all changes touch the same files.
- Repository setup, completed before dispatching Task 1.1:
  - `.apm/` is untracked and listed in `.gitignore`; the change is committed on local `master` as `7930b851f` (`chore: stop tracking APM files and add Worker rules`). Planning documents stay on disk but leave Git tracking.
  - The merged remote branch `origin/copilot/modify-inline-conversion-preview` is deleted.
  - `fix/enter-highlighted-commit` is created from local `master` at `7930b851f`. `origin/master` remains `c4a329c57` until the single push in Task 2.1.
- The first Bazel build is slow. `bazelisk` is installed at `/tmp/opencode/bazelisk`; run long test commands in the background and keep the log path in the report. Do not install extra system packages unless a test actually needs them.
- Single-push rule: only Task 2.1 pushes. If CI fails after the push, report the failure and ask the User before pushing again, because another push triggers another Windows build.
- Likely blockers: the `PageDown` test depends on the candidate page size; the focused path may be bypassed by the keymap command chosen in `Suggestion` state; the Windows release job queues behind the build job and may take over an hour.
- Stage 1 verification: the six session scenarios and the keymap assertions pass, and the diff contains only the session/keymap files and their tests. Stage 2 verification: Linux CI green, a `win-x64-<sha12>` release with `Mozc64_x64.msi` exists, and the User accepts the behaviour on Windows.
- Integration points: keymap TSV parsing in `src/session/keymap.cc` and `src/session/keymap.h`, command dispatch in `Session::SendKeyCompositionState`, focused-candidate selection in `Session::MaybeCommitFocusedSuggestion`, and the converter's `SUGGESTION` state with `PREDICTION` request type.
- No work is left for a later session. Upstream Mozc behaviour and mobile keymaps are out of scope.

## Stage 1: Lock Contract and Implement Enter/Shift+Enter Split

### Task 1.1: Encode the contract as tests and diagnose the raw-commit fallback

- **Agent:** session-agent
- **Objective:** Add tests that express the agreed key contract through the real keymap and session dispatch, then determine why Enter currently commits raw composition instead of a candidate.
- **Dependencies:** None
- **Output:** Updated `src/session/session_test.cc`, `src/session/keymap_test.cc` and `src/session/keymap.h` (the `COMMIT_RAW` enumerator only); a Task Report confirming the two known failures, the added Shift+Enter/ASCII/no-menu coverage, the command resolved for `Enter`, the candidate window category, the focused index, and whether `MaybeCommitFocusedSuggestion` is reached.
- **Validation:** The test targets compile. Run `/tmp/opencode/bazelisk test //session:session_test //session:keymap_test -c dbg --test_output=errors` and record each new test as pass or fail. `EnterCommitsFocusedSuggestionAfterNavigation` and `EnterCommitsRawCompositionForAlphanumericInput` already fail on `master`; the new Shift+Enter tests are expected to fail until Task 1.2. The report must separate measured facts from the installed-build hypothesis.
- **Guidance:** The desktop path is: `EngineConverter` state `SUGGESTION`, request category `PREDICTION`, `Suggestion` keymap rows, then `Session::SendKeyCompositionState`. On `master`, `Suggestion Enter` resolves to `CommitFirstSuggestion`, which is page-local through `CandidateList::MoveToPageIndex(0)`; that is why Down + Enter commits 候補0 while PageDown + Enter passes. Confirm this by reading the code and the test result rather than assuming it. Do not change production behaviour in this Task. The keymap tests need `CompositionState::COMMIT_RAW` to compile, so add that enumerator and its comment in this Task; do not register `CommitRaw` or change any binding until Task 1.2.

1. Read `src/session/session.cc` (`SendKeyCompositionState`, `MaybeCommitFocusedSuggestion`, `MaybeCommitSuggestionByNumber`, `CommitFirstSuggestion`), `src/session/keymap.cc`, and the `Suggestion` sections of the four desktop keymaps.
2. Review the four existing session tests, then extend or add so all six contract scenarios are covered, using the existing helpers `InitSessionToPrecomposition`, `InsertCharacterChars`, `SendKey`, and `SendSpecialKey`.
3. In the tests, assert `candidate_window().category()` and `focused_index()` at the point where the menu is open, so a category or focus mismatch is visible.
4. Add keymap tests asserting `Suggestion Enter` resolves to `CompositionState::COMMIT` and `Suggestion Shift Enter` to `CompositionState::COMMIT_RAW` for `MSIME`, `ATOK`, `KOTOERI`, and `CHROMEOS`.
5. Run the two test targets. Record every failing test name, the expected and actual result, and the exact production path that produced it.
6. Confirm whether Enter resolves to `Commit`, `CommitFirstSuggestion`, or another command under `SUGGESTION`; whether `MaybeCommitFocusedSuggestion` returns false and which guard rejects it; and how `MoveToPageIndex` makes `CommitFirstSuggestion` page-local. Record the candidate window category and focused index observed in the tests.
7. Remove any temporary debug logging. Commit the tests and the `COMMIT_RAW` enumerator with `test: cover Enter and Shift+Enter suggestion commits` and write the Task Report.

### Task 1.2: Add the raw-commit command and invert the key bindings

- **Agent:** session-agent
- **Objective:** Make the Task 1.1 tests pass by routing Enter to the focused-candidate commit and Shift+Enter to a new raw-commit command.
- **Dependencies:** Task 1.1 (Same-Worker)
- **Output:** Changes to `src/session/keymap.h`, `src/session/keymap.cc`, `src/session/session.cc`, `src/session/session.h` if needed, and `src/data/keymap/{ms-ime,atok,kotoeri,chromeos}.tsv`; all contract tests passing.
- **Validation:** `/tmp/opencode/bazelisk test //session:session_test //session:keymap_test //engine:engine_converter_test -c dbg --test_output=errors` passes. `git diff --stat` touches only the files above. Changed C++ files are clean under the repository `.clang-format`.
- **Guidance:** Follow the Spec's "Implementation Design" exactly. `COMMIT_RAW` must call raw `Commit` and must never read the candidate window. `MaybeCommitFocusedSuggestion` keeps the composition, `SUGGESTION` and ASCII guards, drops the hard `PREDICTION` category check, and excludes mobile through `context_->GetRequest().mixed_conversion()` and `zero_query_suggestion()`. Keep number shortcuts, Tab/Space navigation, rendering and mobile keymaps unchanged. If Task 1.1 found a concrete cause that differs from the Spec's assumption, fix that cause and explain it in the report.

1. Keep the `COMMIT_RAW` enumerator and comment added in Task 1.1; do not add a second value.
2. Register `CommitRaw` in `KeyMapManager::InitCommandData` and handle it in `Session::SendKeyCompositionState` as `return Commit(command);`.
3. Change `Suggestion Enter` to `Commit` and `Suggestion Shift Enter` to `CommitRaw` in the four desktop keymaps; remove the replaced `CommitFirstSuggestion` rows. Leave `Ctrl Enter` bindings and `mobile.tsv` untouched.
4. Adjust `MaybeCommitFocusedSuggestion` per the Spec so a desktop candidate window with a focused index commits that candidate, regardless of whether the category is `PREDICTION` or `SUGGESTION`.
5. Update comments that still describe Enter as committing the composition itself or Shift+Enter as committing the first suggestion.
6. Run the three test targets, fix failures within the Task's scope, and run `clang-format` on changed C++ files.
7. Commit with `fix: commit highlighted suggestion on Enter and raw on Shift+Enter` and write the Task Report with the test command and result summary.

## Stage 2: Verify, Release and Acceptance

### Task 2.1: Independent verification, single push and Windows release

- **Agent:** release-agent
- **Objective:** Verify the branch independently, merge it into `master`, push once, and confirm the Linux CI and Windows x64 release results.
- **Dependencies:** Task 1.2 by session-agent (Cross-Worker)
- **Output:** A verification report containing test results, diff review, merge commit, push confirmation, Linux CI run URL and result, Windows workflow run URL and result, release tag and asset URL, and a short Windows acceptance checklist for the User.
- **Validation:** All three test targets pass on the branch. `git status` is clean. The merge into `master` is a `--no-ff` merge with no conflicts. The single push succeeds. Linux CI turns green; the Windows workflow publishes `win-x64-<sha12>` with `Mozc64_x64.msi`. The Task then pauses for the User to install the MSI and confirm the five acceptance cases.
- **Guidance:** Do not change production code or tests. If verification finds a defect, report it back instead of fixing it. Do not push before every local check passes, and do not push twice without the User's approval. Use `gh` (already authenticated) for run and release status.

1. Fetch `origin`, check out `fix/enter-highlighted-commit`, and confirm the tip matches the Task 1.2 report.
2. Run `/tmp/opencode/bazelisk test //session:session_test //session:keymap_test //engine:engine_converter_test -c dbg --test_output=errors` from `src/`.
3. Review `git diff master...HEAD` against `.apm/spec.md`: command registration, TSV rows, guarded focused commit, unchanged `mobile.tsv`, unchanged number/navigation behaviour.
4. Confirm `.apm/` is untracked and ignored, then merge with `git checkout master && git merge --no-ff fix/enter-highlighted-commit`.
5. Push `master` once. Record the pushed commit SHA.
6. Watch the Linux CI and Windows workflows with `gh run list` / `gh run watch`; record their URLs and conclusions.
7. Confirm the release with `gh release view win-x64-<sha12>` and the `Mozc64_x64.msi` asset.
8. Report the release URL and pause for the User's Windows acceptance: `hashiru` + Enter inserts 走る; Down + Enter inserts the new highlight; PageDown + Enter inserts the second-page highlight; Shift+Enter inserts はしる; A mode Enter and Shift+Enter insert typed ASCII.
