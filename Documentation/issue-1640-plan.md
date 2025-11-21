# Issue #1640 – tvOS Series Focus Plan

## Goal & Context
- Resolve the Season → Episode focus loss that occurs on tvOS 18.5 (and potentially tvOS 26) when the “Next Up” item is off screen and `EpisodeHStack` jumps between placeholder/loading cards.
- Keep the plan scoped to the `feature/1640-tvos-focus` branch and bundle-only code (`Swiftfin tvOS/Views/ItemView/Components/EpisodeSelector/...`).
- Maintain backwards-compatible behavior for empty/error/loading states while ensuring the `FocusGuide` always lands on a visible card.

## Baseline & Investigation
1. **Environment prep** – Stay on `feature/1640-tvos-focus`, run `carthage update --use-xcframeworks`, `swiftformat .`, and build `Swiftfin tvOS` in Xcode 16.4 to mirror CI.  
2. **Hardware verification** – Install the build on:
   - tvOS 26 hardware (prefer 3rd‑gen Apple TV 4K + any older model still on the same OS).
   - tvOS 18.5 hardware (baseline reproduction device).  
   Use a series with a “Next Up” card that sits mid-season so the first focus request is off screen.
3. **Logging** – Temporarily add `Logger(seriesEpisodeFocus:)` statements (or wrap existing `log.debug` helpers) around `getContentFocus()`, `.onChange` handlers, and `proxy.scrollTo` in `EpisodeHStack`. Capture logs with `Console.app` or `log stream --predicate 'process == "Swiftfin"'`.
4. **Document matrix** – Note whether the bug exists on tvOS 26. If it does not repro there, explicitly call out that 18.5 is the only affected OS so we can gate the fix if needed.

## Implementation Strategy
1. **Focus lifecycle coordinator (EpisodeHStack.swift)**  
   - Introduce a lightweight helper (either a nested `EpisodeFocusCoordinator` struct or new `@State` bundle) that holds:
     - `pendingEpisodeID` – the target focus once layouts finish.
     - `lastCommittedEpisodeID` – mirrors `focusedEpisodeID` but only when the element still exists.
     - `isApplyingFocus` flag to guard re-entrancy.  
   - Add methods `queueFocus(for state: SeasonItemViewModel.State)` and `consumePendingFocus(after elementsReady:)` to centralize the logic currently spread across `getContentFocus()`, `.onChange` blocks, and the `DispatchQueue.main.asyncAfter` call.  
   - Keep `focusedEpisodeID` writes inside a single helper so every path (empty/error/loading/content) runs through the guard + visibility checks.
2. **Defer writes until CollectionHStack is ready**  
   - Replace direct `focusedEpisodeID = ...` assignments with:
     ```swift
     withTransaction(Transaction(animation: .none)) { txn in
         txn.disablesAnimations = true
         DispatchQueue.main.async {
             guard !isApplyingFocus else { return }
             isApplyingFocus = true
             focusedEpisodeID = coordinator.nextFocusID(for: viewModel.elements, state: viewModel.state)
             isApplyingFocus = false
         }
     }
     ```  
   - The helper should no-op if the requested episode id is already focused or if the ID is no longer part of `viewModel.elements`. This stabilizes the `FocusGuide` callback that fires repeatedly while the stack is rebuilding.
3. **Season-change coordination**  
   - In `.onChange(of: viewModel.id)` set `pendingEpisodeID = playButtonItem?.id ?? viewModel.elements.first?.id` and delay consumption until `.content` renders.  
   - Observe `viewModel.elements` (or `viewModel.state` entering `.content`) to trigger a single `proxy.scrollTo` and focus apply:
     ```swift
     .onChange(of: viewModel.state) { _, newValue in
         guard newValue == .content else { return }
         coordinator.consumePendingFocus(availableIDs: viewModel.elements.map(\\.id))
     }
     ```  
   - When `lastFocusedEpisodeID` is invalidated (episode removed / season switch), immediately fall back to `viewModel.elements.first?.id` to avoid focus requests against stale IDs.
4. **Configurable scroll timing**  
   - Replace the hard-coded `0.1` delay in `contentView` with a static constant (`EpisodeHStack.playButtonScrollDelay`).  
   - If tvOS 18 hardware still needs a longer wait, bump only for `UIDevice.current.platformGeneration < 3` to keep newer units responsive. Expose the constant at the top of the file for future tuning.
5. **Guard rails & logging**  
   - Add a dedicated `Logger` to surface cases where the focus request was dropped because no visible element matched.  
   - Consider gating the entire workaround via `if #available(tvOS 19, *)` if tvOS 26 ultimately proves unaffected; until then, keep the path universal but clearly comment why the guard exists.

## Testing Plan
1. **Devices / OS**  
   - tvOS 26: verify season picker → episode hstack focus, including empty/error/loading transitions.  
   - tvOS 18.5: reproduce pre-fix behavior, apply fix, confirm focus now lands on the “Next Up” card or fallback episode.  
2. **Simulators**  
   - Smoke test on 18.4 and 26 simulators to ensure the new guard does not cause build-time or runtime crashes.
3. **Regression scenarios**  
   - Empty season (should focus the empty card only once).  
   - Error state (force `.error` via network toggle) and confirm focus returns, then successfully transitions back to `.content`.  
   - Continue watching with “Next Up” near the middle of the list, plus manual season hops (1 → 3 → 2) to confirm `pendingEpisodeID` tracking works.  
   - Trigger downloads/experimental toggles that rebuild `SeasonItemViewModel` to ensure focus coordinator resets correctly.
4. **Build validation**  
   - Run `fastlane buildLane scheme:"Swiftfin tvOS"` plus `swiftformat .` / `swiftlint` as usual before opening the PR.  
   - Revert temporary logging once confidence is gained, or leave guarded by `#if DEBUG`.

## PR Checklist
1. Branch stays `feature/1640-tvos-focus`; keep the issue number in commits for traceability.  
2. Split commits into:
   - Focus coordinator + EpisodeHStack logic changes.
   - Timing/delay constants and optional logging cleanup.  
3. PR description:
   - Outline the focus race, hardware tested, and whether the issue still affects tvOS 26.  
   - Include a testing grid (devices/OS/states) and finish with “Closes #1640.”  
4. Labels: `tvOS`, `bug`. Request review from `@LePips` and `@JPKribs`.  
5. Follow up: only cherry-pick to a release branch if maintainers ask; otherwise rely on `main` → TestFlight automation.
