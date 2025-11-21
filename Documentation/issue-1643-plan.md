# Issue #1643 – iOS Episode Picker Refresh Plan

## Goal
Eliminate the placeholder flash when exiting video playback by refreshing only the data that changes (user data) rather than clearing and rebuilding entire season view models. Implement a maintainable flow ahead of PR #1581, but compatible with its upcoming changes.

## Proposed Approach
1. **New BaseItemDto utilities**
   - Extend `BaseItemDto` with `refresh()` and `refreshUserData()` helpers (building on the sketch in the issue discussion) using Factory’s `currentUserSession`.
   - `refresh()` fetches the full item (for explicit refresh actions).
   - `refreshUserData()` only calls `Paths.getItemUserData` and merges the result into `userData`.

2. **MediaPlayerManager / Episode queue updates**
   - In `MediaPlayerManager` (and `EpisodeMediaPlayerQueue`), replace current `.send(.backgroundRefresh)`/`.send(.refresh)` calls triggered on playback end with:
     1. `item.refreshUserData()` to update progress/favorite state.
     2. A targeted `SeriesItemViewModel` update that only refreshes user data for `playButtonItem`/season episodes instead of clearing `seasons`.
   - Ensure the injected `SeriesItemViewModel` (used by Episode queue overlays) exposes a method to “soft refresh” user data without resetting `seasons`.

3. **SeriesItemViewModel refactor**
   - Introduce a `refreshUserDataOnly()` action that:
     - Iterates through `seasons` and updates each `SeasonItemViewModel`’s episodes with fresh `userData` but leaves their `state` intact.
     - Updates `playButtonItem`’s `userData` so watch indicators stay accurate.
   - Reserve `.refresh` for full metadata reload initiated by pull-to-refresh or ItemEditor.

4. **Guard EpisodeSelector**
   - Ensure `SeriesEpisodeSelector` responds to `refreshUserDataOnly` by updating progress indicators without showing placeholder. No placeholder should appear unless a season truly enters `.initial` (e.g., first load, explicit refresh).

## Testing Plan
1. **Reproduction baseline (current main)**
   - iOS 18.5 simulator + physical devices (e.g., iPhone 15 Pro): open series, play, exit, confirm placeholder bug occurs.
   - Repeat on iOS 26 (beta) to document slower-path behavior.
2. **After fix**
   - Same devices: verify exiting player keeps the episode grid intact, updates progress indicators, and no flicker occurs.
   - Test switching seasons quickly, pull-to-refresh, and actions that legitimately call `.refresh`.
   - Ensure tvOS EpisodeSelector (shared logic) still works.
3. **Automation**
   - Build via `fastlane buildLane scheme:"Swiftfin"`; ensure `swiftformat .` passes.

## PR Checklist
1. Branch: `feature/1643-ios-episode-picker` (already created).
2. Commits:
   - Add `BaseItemDto` refresh helpers.
   - Update MediaPlayerManager/Episode queue to use user data refresh.
   - Add `SeriesItemViewModel` user-data-only refresh path and hook up EpisodeSelector.
3. PR Description:
   - Reference issue #1643 and #1589/#1581 context.
   - Summaries of new helpers and series refresh behavior.
   - Testing matrix (devices/OS versions).
4. Labels: `iOS`, `bug`, request reviews from `@JPKribs` and `@LePips`.
