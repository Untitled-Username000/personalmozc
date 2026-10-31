# APM Spec

## Overview

This APM session fixes the Enter and Shift+Enter behaviour while the inline prediction menu is open in `personalmozc`, a Windows-focused fork of Mozc. Enter must commit the candidate currently highlighted in the menu, including after Up/Down and PageUp/PageDown navigation, instead of committing the raw hiragana composition. Shift+Enter must commit the raw composition. Success means the session and keymap tests encode this contract, Linux CI passes, a new Windows x64 MSI is released, and the User confirms the behaviour in the installed IME.

## Workspace

- Working repository: `/workspaces/personalmozc`, one Git repository, fork of `google/mozc`.
  - `origin`: `https://github.com/Untitled-Username000/personalmozc` (write target).
  - `upstream`: `https://github.com/google/mozc.git` (read-only reference).
  - Base branch: `master`. `origin/master` is `c4a329c57`; local `master` is at `7930b851f`, the repository setup commit `chore: stop tracking APM files and add Worker rules`.
- Working target for this session: Windows x64 build and the desktop keymaps. Source changes stay in the shared C++/TSV tree.
- Read-only reference: upstream Mozc history and the existing fork commits on the inline-suggestion feature line.
- Authoritative sources:
  - The current code and its tests, especially `src/session/session.cc`, `src/session/session.h`, `src/session/keymap.cc`, `src/session/keymap.h`, `src/engine/engine_converter.cc`, `src/engine/engine_output.cc`, `src/data/keymap/*.tsv`, `src/session/session_test.cc`, `src/session/keymap_test.cc`.
  - This session's User requirements (the key-binding contract below).
  - `.apm/archives/session-2026-10-31-001/` contains empty templates only and is not authoritative.
- `AGENTS.md` exists at the workspace root and contains the `APM_RULES` block.
- `.apm/` is untracked and listed in `.gitignore`.
- Branches: `fix/enter-highlighted-commit`, created from local `master` at `7930b851f`, for this work. Commit messages use `fix: <description>` and `test: <description>`.
- Delivery: work is committed on the branch only. The Manager merges into `master` and pushes once, after all work and local validation are complete, to avoid triggering intermediate builds. No pull request.

## Notes

- The User sees Enter close the menu and leave hiragana. That is the raw `Session::Commit()` fallback, so the focused-candidate path is either not reached or returns false in the installed build.
- The installed MSI may predate commit `519eaf543`, which added `Suggestion Enter CommitFirstSuggestion`. Task 1.1 must establish whether current `master` already commits a candidate. If it does, the installed build is stale; the tests still lock the contract and the Shift+Enter split is still required.
- Verified on `master` `c4a329c57` on 2026-10-31 with `bazelisk test //session:session_test -c dbg`: the suite is sharded into 8 runs and reports two failures.
  - `EnterCommitsFocusedSuggestionAfterNavigation` commits 候補0 instead of the highlighted 候補1.
  - `EnterCommitsRawCompositionForAlphanumericInput` commits Ａ instead of the typed A.
  - `EnterCommitsFocusedSuggestionWhileTyping` and `EnterCommitsFocusedSuggestionOnSecondPage` pass.
- The passing second-page test exposes a subtlety: `CommitFirstSuggestion` calls `CandidateList::MoveToPageIndex(0)`, which is page-local. It commits the first candidate of the **current page**, not the globally first candidate. So PageDown + Enter happens to work only when the first item of that page is the intended one; Up/Down within a page and any other highlight commit the wrong candidate. This matches the reported "Enter does not commit the highlighted item" behaviour. A build without the `Suggestion Enter` binding would instead take the raw `Commit` path, which matches the reported hiragana output.
- The desktop prediction window is requested as `ConversionRequest::PREDICTION` while the converter state remains `SUGGESTION`. Do not rely on the state alone to decide the window category; assert the actual category in tests.
- The local development environment is Linux. Windows UI behaviour cannot be exercised here; the User validates the released MSI.
- Bash/zsh environment date: 2026-10-31.

## Key Binding Contract

Applies while the prediction menu is displayed during kana composition on desktop.

| Situation | `Enter` | `Shift+Enter` |
|---|---|---|
| Menu open, any candidate highlighted (first, or after Up/Down/PageUp/PageDown) | Commit that highlighted candidate | Commit the raw composition (e.g. はしる) |
| Menu open, ASCII/alphanumeric input mode (`HALF_ASCII` or `FULL_ASCII`, language-bar "A") | Commit raw text | Commit raw text |
| No candidate menu | Commit raw composition | Commit raw composition |
| Mobile keymaps (`mobile.tsv`) | Unchanged | Unchanged |

The raw composition is the text the User typed, transliterated by the current input mode (hiragana in the reported case).

## Current Behaviour and Root Path

- `Session::SendKeyCompositionState` chooses `KeyMapManager::GetCommandSuggestion` when `EngineConverter::CheckState(SUGGESTION)` is true, otherwise `GetCommandComposition`.
- `EngineConverter::Suggest` sets `state_ = SUGGESTION` but, on desktop (`!mixed_conversion && !zero_query_suggestion`), sets `request_type_ = PREDICTION`. `FillOutput` therefore reports `commands::PREDICTION` with a focused index.
- `Composition Enter` maps to `Commit`. `Session::Commit` handling calls `MaybeCommitFocusedSuggestion`, which commits the candidate whose global `index()` equals `focused_index`, then falls back to raw `Session::Commit`.
- `Suggestion Enter` currently maps to `CommitFirstSuggestion`, which calls `CandidateList::MoveToPageIndex(0)` and commits the first candidate **of the current page**, regardless of which candidate is highlighted.
- `Suggestion Shift Enter` is also `CommitFirstSuggestion` after `519eaf543`. It therefore ignores the ASCII guard and can commit a candidate in A mode.
- `CommitFirstSuggestion` ignores the boolean result of `CommitSuggestionByIndex`; on failure it still moves to `PRECOMPOSITION`, which can look like the menu closing without a conversion.
- `MaybeCommitFocusedSuggestion` currently returns false when `candidate_window.category() != commands::PREDICTION` or `focused_index` is absent. A category mismatch or a missing focus is the most likely reason the focused path never runs in the installed build.

## Implementation Design

### Raw commit command

- Add `COMMIT_RAW` to `keymap::CompositionState::Commands` in `src/session/keymap.h` with a comment: commit the composition without consulting any suggestion or candidate.
- Register it in `KeyMapManager::InitCommandData` (`src/session/keymap.cc`) with the command string `CommitRaw`.
- Handle it in `Session::SendKeyCompositionState` (`src/session/session.cc`) as `return Commit(command);`.
- The command must never read the candidate window and must never call `CommitFirstSuggestion` or `MaybeCommitFocusedSuggestion`.

### Keymap bindings

For `ms-ime.tsv`, `atok.tsv`, `kotoeri.tsv`, and `chromeos.tsv` in the `Suggestion` section:

- `Suggestion Enter` -> `Commit`
- `Suggestion Shift Enter` -> `CommitRaw`
- Remove the current `Suggestion Enter CommitFirstSuggestion` and `Suggestion Shift Enter CommitFirstSuggestion` rows in those files.
- Leave any `Suggestion Ctrl Enter CommitFirstSuggestion` row as is.
- Leave `mobile.tsv` unchanged.
- Leave `Composition Enter Commit` and `Composition VirtualEnter Commit` unchanged.

### Focused commit robustness

In `MaybeCommitFocusedSuggestion` (`src/session/session.cc`):

- Keep the guards for `context_->state() == ImeContext::COMPOSITION`, `CheckState(SUGGESTION)`, and `IsAsciiInputMode()`.
- Replace the hard `category() != commands::PREDICTION` rejection with a desktop check based on the request: return false when `context_->GetRequest().mixed_conversion()` or `context_->GetRequest().zero_query_suggestion()` is true.
- Require `candidate_window.has_focused_index()` and a matching candidate entry.
- Accept both `commands::PREDICTION` and `commands::SUGGESTION` categories on desktop. `CommitSuggestionAtPageIndex` already accepts both.
- If diagnosis in Task 1.1 finds a different concrete cause, record it and fix that cause while preserving the contract and the mobile exclusion above.

### Invariants

- `MaybeCommitSuggestionByNumber` (1-9 shortcuts) keeps its current behaviour, including the ASCII guard and the full-width digit normalization.
- `Tab`, `Space`, `Down`, `Up`, `PageUp`, and `PageDown` keep their current navigation behaviour.
- The expanded prediction rendering, footer, and index display do not change.
- No engine-side conversion changes unless the diagnosis proves one is required; if so, document the reason in the Task Report.
- `CommitFirstSuggestion` remains available for existing bindings such as ATOK `Ctrl Enter`.
- Mobile (`mixed_conversion` or `zero_query_suggestion`) behaviour must not change.

## Testing and Validation

Add or update tests in `src/session/session_test.cc` covering the contract through the real keymap and session dispatch:

1. Enter with the first candidate focused commits that candidate (`候補0`), not the raw composition.
2. Down then Enter commits the highlighted second candidate (`候補1`).
3. PageDown then Enter commits the highlighted candidate on the second page (`候補8`).
4. Shift+Enter with the menu open commits the raw composition (`あいうえお`), not a candidate.
5. Enter and Shift+Enter in ASCII input mode commit raw text.
6. Enter with no candidate window commits the raw composition.

Add or update tests in `src/session/keymap_test.cc` asserting, for `MSIME`, `ATOK`, `KOTOERI`, and `CHROMEOS`:

- `Suggestion Enter` resolves to `CompositionState::COMMIT`.
- `Suggestion Shift Enter` resolves to `CompositionState::COMMIT_RAW`.

Keep `CommitFirstSuggestion` coverage for bindings that still use it.

Validation commands, run from `src/`:

```sh
/tmp/opencode/bazelisk test //session:session_test //session:keymap_test //engine:engine_converter_test -c dbg --test_output=errors
```

Formatting follows the repository `.clang-format`; changed C++ files must be formatted.

## Delivery and Acceptance

- Worker commits stay on `fix/enter-highlighted-commit`.
- After verification, the Manager merges into `master` and pushes once. The push triggers Linux CI and the Windows x64 MSI release workflow.
- The release workflow publishes tag `win-x64-<sha12>` and asset `Mozc64_x64.msi` on the default branch.
- The User installs that MSI and confirms:
  1. Type `hashiru`, 走る is highlighted, Enter inserts 走る.
  2. Down highlights another candidate, Enter inserts that candidate.
  3. PageDown then Enter inserts the highlighted candidate on that page.
  4. Shift+Enter inserts はしる.
  5. In A mode, Enter and Shift+Enter insert the typed ASCII text.
- If the User's check fails, reopen the analysis with the exact built commit and symptom.
