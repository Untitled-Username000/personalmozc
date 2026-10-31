---
agent: session-agent
stage: 1
task: 1
log_path: ".apm/memory/stage-01/task-01-01.log.md"
---

# Task 1.1 - Encode the contract as tests and diagnose the raw-commit fallback

## Objective

Add tests that express the agreed Enter/Shift+Enter contract through the real desktop keymap and session dispatch, then determine why Enter currently commits the raw composition instead of the focused candidate.

## Instructions

### The contract to encode

Applies while the inline prediction menu is displayed during kana composition on desktop:

| Situation | `Enter` | `Shift+Enter` |
|---|---|---|
| Menu open, any candidate highlighted (first, or after Up/Down/PageUp/PageDown) | Commit that highlighted candidate | Commit the raw composition (e.g. はしる) |
| Menu open, ASCII/alphanumeric input mode (`HALF_ASCII` or `FULL_ASCII`, language-bar "A") | Commit raw typed text | Commit raw typed text |
| No candidate menu | Commit raw composition | Commit raw composition |

The raw composition is the text the User typed, transliterated by the current input mode (hiragana in the reported case). Mobile keymaps are out of scope and must not change.

### What the code does today (verify this, do not assume it)

- `Session::SendKeyCompositionState` chooses `KeyMapManager::GetCommandSuggestion` when `EngineConverter::CheckState(SUGGESTION)` is true, otherwise `GetCommandComposition`.
- `EngineConverter::Suggest` sets `state_ = SUGGESTION` but, on desktop (`!mixed_conversion && !zero_query_suggestion`), sets `request_type_ = PREDICTION`. `FillOutput` therefore reports `commands::PREDICTION` with a focused index.
- `Composition Enter` maps to `Commit`. `Session::Commit` handling calls `MaybeCommitFocusedSuggestion`, which commits the candidate whose global `index()` equals `focused_index`, then falls back to raw `Session::Commit`.
- `Suggestion Enter` currently maps to `CommitFirstSuggestion`, which calls `CandidateList::MoveToPageIndex(0)` and commits the first candidate **of the current page**, regardless of which candidate is highlighted.
- `Suggestion Shift Enter` is also `CommitFirstSuggestion`, so it ignores the ASCII guard and can commit a candidate in A mode.
- `CommitFirstSuggestion` ignores the boolean result of `CommitSuggestionByIndex`; on failure it still moves to `PRECOMPOSITION`, which can look like the menu closing without a conversion.
- `MaybeCommitFocusedSuggestion` currently returns false when `candidate_window.category() != commands::PREDICTION` or `focused_index` is absent.

Existing tests in `src/session/session_test.cc` cover four of the six cases:

- `EnterCommitsFocusedSuggestionWhileTyping` (passes today: first candidate is also the page's first candidate).
- `EnterCommitsFocusedSuggestionAfterNavigation` (fails today: commits 候補0 instead of 候補1 after Down).
- `EnterCommitsFocusedSuggestionOnSecondPage` (passes today, but only because `CommitFirstSuggestion` is page-local and the focused candidate is the page's first).
- `EnterCommitsRawCompositionForAlphanumericInput` (fails today: commits Ａ instead of the typed A).

### Steps

1. Read `src/session/session.cc` (`SendKeyCompositionState`, `MaybeCommitFocusedSuggestion`, `MaybeCommitSuggestionByNumber`, `CommitFirstSuggestion`), `src/session/keymap.cc`, and the `Suggestion` sections of `src/data/keymap/ms-ime.tsv`, `atok.tsv`, `kotoeri.tsv`, `chromeos.tsv`, and `mobile.tsv` (mobile for contrast only).

2. Review the four existing tests above, keep them, and extend or adjust them so all six contract scenarios are covered:

   1. Enter with the first candidate focused commits `候補0`, not the raw composition.
   2. Down then Enter commits the highlighted second candidate `候補1`.
   3. PageDown then Enter commits the highlighted candidate on the second page `候補8`.
   4. Shift+Enter with the menu open commits the raw composition `あいうえお`, not a candidate.
   5. Enter and Shift+Enter in ASCII input mode commit the typed ASCII text `A`; currently Enter alone is covered, so extend it to Shift+Enter or add a separate case.
   6. Enter with no candidate window commits the raw composition.

   Use the existing helpers `InitSessionToPrecomposition`, `InsertCharacterChars`, `SendKey`, and `SendSpecialKey`, and match the surrounding test style (`MockEngine`, `MockConverter`, `StartPrediction`, `CommitSegmentValue`, `FinishConversion`).

3. In each session test, assert `command.output().candidate_window().category()` and `focused_index()` at the point where the menu is open, so a category or focus mismatch is visible. Record the observed values for the diagnosis.

4. Add keymap tests in `src/session/keymap_test.cc` asserting, for `MSIME`, `ATOK`, `KOTOERI`, and `CHROMEOS`:

   - `Suggestion Enter` resolves to `CompositionState::COMMIT`.
   - `Suggestion Shift Enter` resolves to `CompositionState::COMMIT_RAW`.

   Follow the existing fixture style, for example `KeyMapManager manager(GetDefaultConfig(config::Config::MSIME));` with `KeyParser::ParseKey` and `GetCommandSuggestion`. Keep existing `CommitFirstSuggestion` coverage such as ATOK `Ctrl Enter` and `ZeroQuerySuggestion`.

   To make those tests compile, add `COMMIT_RAW` to `CompositionState::Commands` in `src/session/keymap.h` with a comment saying it commits the composition without consulting any suggestion or candidate. Do not register `CommitRaw` in `keymap.cc`, do not change any `.tsv` binding, and do not change `Session` dispatch: registration, dispatch and bindings belong to a later Task, so behaviour must not change here.

5. Run the two test targets from `/workspaces/personalmozc/src`:

   ```sh
   /tmp/opencode/bazelisk test //session:session_test //session:keymap_test -c dbg --test_output=errors
   ```

   Record every new or changed test as pass or fail, with expected and actual values. The suite is sharded, so report the per-test result rather than only the target summary. `EnterCommitsFocusedSuggestionAfterNavigation`, `EnterCommitsRawCompositionForAlphanumericInput`, both new Shift+Enter session cases, and all eight keymap assertions are expected to fail until the later implementation Task; the first-candidate, second-page and no-menu cases are expected to pass. If a result differs from that expectation, record it.

6. Diagnose and record, as measured facts:

   - Whether `Enter` under `SUGGESTION` resolves to `Commit`, `CommitFirstSuggestion`, or another command.
   - Whether `MaybeCommitFocusedSuggestion` is reached and, if it returns false, which guard rejects it.
   - The candidate window `category()` and `focused_index()` observed in each test.
   - How `CandidateList::MoveToPageIndex(0)` makes `CommitFirstSuggestion` page-local and why PageDown + Enter can pass by accident.

   Separate these measured facts from the hypothesis that the installed MSI predates the current code. If you add temporary debug output, remove it before committing.

7. Run `clang-format` on the changed C++ files. Commit the tests and the enumerator on the current branch with:

   ```sh
   git add src/session/session_test.cc src/session/keymap_test.cc src/session/keymap.h
   git commit -m "test: cover Enter and Shift+Enter suggestion commits"
   ```

   Stage only these files. Do not stage `.apm/`. Do not push.

## Workspace

- Repository: `/workspaces/personalmozc`. Branch `fix/enter-highlighted-commit` is already checked out in the main directory. No worktree is used.
- Base branch is `master`; do not merge into it.
- Run all commands from `/workspaces/personalmozc/src`.
- Every `.apm/` path refers to `/workspaces/personalmozc/.apm/`.
- Use `/tmp/opencode/bazelisk` as the Bazel launcher. The first build is slow; run it in the background if needed and record the log path.

## Output

- Updated `src/session/session_test.cc`, `src/session/keymap_test.cc`, and `src/session/keymap.h` (the `COMMIT_RAW` enumerator only), committed on `fix/enter-highlighted-commit`.
- A Task Log at `.apm/memory/stage-01/task-01-01.log.md` and a Task Report in `.apm/bus/session-agent/report.md`, using the formats in `.agents/apm-guides/task-logging.md` §5.1 and §5.2. Clear `task.md` after processing it.
- The report must confirm: the two known failures, the added Shift+Enter/ASCII/no-menu coverage, the command resolved for `Enter`, the candidate window category, the focused index, whether `MaybeCommitFocusedSuggestion` is reached, and the commit SHA.

## Validation

- `//session:session_test` and `//session:keymap_test` compile and run with the command above; every new test result is recorded.
- The production behaviour is unchanged in this Task: no command registration, no keymap binding change, no `Session` dispatch change. The only production edit is the unused `COMMIT_RAW` enumerator.
- No changes to `mobile.tsv`, number shortcuts, Tab/Space navigation, candidate rendering, or files outside the list above.
- Changed C++ files are formatted with the repository `.clang-format`, and `git status` shows only the intended files committed.
