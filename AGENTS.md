# Swiftfin Agent Field Guide

## Key References
- [Contribution Guide](Documentation/contributing.md) – setup steps, linting and localization rules, and links back to the Jellyfin PR policy.
- [README](README.md) – marketing overview plus current platform minimums (`iOS 16+`, `tvOS 17+`, Jellyfin `10.11`) and TestFlight link (iOS-only, see [discussion #1294](https://github.com/jellyfin/Swiftfin/discussions/1294) for tvOS beta context).
- [Version Policy](Documentation/version.md) – rationale for minimum OS support and how deprecations are communicated.
- [Player Matrix](Documentation/players.md) – feature gaps between the Swiftfin (VLCKit) and Native (AVKit) players.
- [Library Support](Documentation/libraries.md) – current coverage table for Shows, Movies, Collections, Live TV, etc.
- GitHub automation:
  - Issue templates live under `.github/ISSUE_TEMPLATE`.
  - Release notes are grouped via `.github/release.yml`.
  - CI workflows: `.github/workflows/ci.yml`, `lint-pr.yaml`, and `testflight.yml`.

## Branching Strategy & Release Flow
- `main` is the only long-lived branch. All PRs merge directly into `main`, which triggers the “Build 🔨” workflow for both `Swiftfin` (iOS/iPadOS) and `Swiftfin tvOS`.
- Create topic branches from an up-to-date `origin/main`. Team branches are lightweight and named descriptively (`hour-minute-formatting`, `fix-live-tv-media-source`, `featureTemplates`, etc.). Avoid opening PRs straight from your `main` branch as seen in [PR #1816](https://github.com/jellyfin/Swiftfin/pull/1816); it complicates rebasing and follow-up work.
- Keep commits cohesive. Long-running work (example: [PR #1798](https://github.com/jellyfin/Swiftfin/pull/1798)) used WIP commits plus periodic merges from `main` to keep CI green.
- Labels drive automatically generated release notes. Apply at least one of `enhancement`, `bug`, `crash`, or `developer` so your change appears under the right heading, and add platform labels (`iOS`, `tvOS`) when appropriate.

## Pull Request Expectations
1. **Prep environment** – Follow the Contribution Guide: install Xcode 15+ (CI currently runs Xcode 16.4), `brew install carthage swiftformat swiftgen`, and run `carthage update --use-xcframeworks`.
2. **Set signing info** – Copy `XcodeConfig/Shared.xcconfig` into `XcodeConfig/DevelopmentTeam.xcconfig` with your `DEVELOPMENT_TEAM` and optional custom `PRODUCT_BUNDLE_IDENTIFIER`.
3. **Sync dependencies** – Carthage pulls VLCKit 3.5.0 (iOS + tvOS) via the `Cartfile`. Swift Package Manager handles everything else (Jellyfin SDK 0.6.0, Defaults 8.2, Factory 2.5, LNPopupController 3.0.7, VLCUI 0.7.4, etc.; see `Swiftfin.xcodeproj/.../Package.resolved`).
4. **Code style & localization** – Run `swiftformat .` (or the Xcode extension) before committing. Add new localized strings through `Translations/*.lproj` and regenerate with `swiftgen config run` so `Shared/Strings/Strings.swift` stays in sync. Per the guide, experimental features may skip localization but production UI cannot.
5. **Self-test** – Build both schemes locally or run fastlane’s build lane to mirror CI: `fastlane buildLane scheme:"Swiftfin"` and again with `"Swiftfin tvOS"`. This is what `ci.yml` executes on macOS 15 runners.
6. **Document the change** – Most successful PRs follow the informal template adopted in [PRs #1798](https://github.com/jellyfin/Swiftfin/pull/1798) and [#1802](https://github.com/jellyfin/Swiftfin/pull/1802): a `### Summary`, screenshots or videos for UI work, explicit testing notes, and references to GitHub issues (`Closes #1783`, `#1801`, etc.).
7. **Respond to reviews quickly** – Maintainers (primarily `@LePips` and `@JPKribs`) routinely request follow-ups (see [PR #1807](https://github.com/jellyfin/Swiftfin/pull/1807) and [#1806](https://github.com/jellyfin/Swiftfin/pull/1806)). Push updates to the same branch and re-run SwiftFormat before re-requesting review.
8. **Merge requirements** – Wait for both “Build 🔨” (two schemes) and “Lint 🧹” (SwiftFormat) to succeed. At least one maintainer approval is required. TestFlight uploads happen through a `repository_dispatch` that calls `fastlane testFlightLane`—coordinate with maintainers for release builds.

## Issue & Bug Workflow (Observations from the 10 Most Recent Closed Bugs)
- **Templates are enforced** – Issues such as [#1764](https://github.com/jellyfin/Swiftfin/issues/1764) and [#1763](https://github.com/jellyfin/Swiftfin/issues/1763) were actionable because they described detailed reproduction steps, app/server versions, and attached videos/screenshots. Encourage reporters to keep the template intact.
- **Duplicates close fast** – [#1765](https://github.com/jellyfin/Swiftfin/issues/1765), [#1762](https://github.com/jellyfin/Swiftfin/issues/1762), [#1761](https://github.com/jellyfin/Swiftfin/issues/1761), and [#1670](https://github.com/jellyfin/Swiftfin/issues/1670) were immediately marked as duplicates (of #1155 or #787) and linked to the fix branch or discussion (#1294). Always search open/closed issues before filing new ones.
- **Platform context matters** – Several tvOS-specific bugs highlighted gaps in the Native (AVKit) player (subtitles/menu, issues #1762/#1761) compared to the default Swiftfin (VLCKit) player. Use [Documentation/players.md](Documentation/players.md) to explain trade-offs when triaging.
- **Beta OS caveats** – [#1700](https://github.com/jellyfin/Swiftfin/issues/1700) confirmed we do not guarantee support for beta OS builds; ask for TLS 1.2/1.1 usage and direct connections when VLCKit fails, and verify problems still occur on official releases.
- **Server configuration checks** – [#1687](https://github.com/jellyfin/Swiftfin/issues/1687) (Live TV) and [#1681](https://github.com/jellyfin/Swiftfin/issues/1681) (stuck spinner) resolved after reconfiguration/reinstall. Prompt reporters to confirm server tuners, restart the app after toggling experimental features, or fully delete the app rather than relying on offloaded installs.
- **Misfiled tickets are closed politely** – [#1815](https://github.com/jellyfin/Swiftfin/issues/1815) turned out to be Streamyfin, not Swiftfin. Double-check the client name/version before filing to avoid churn.
- **Communication style** – Maintainers reference discussions, provide interim TestFlight links (issue #1764), and use checklists or thumbs-up reactions to confirm fixes. Mirror that tone when responding.

## Guidance for Different Platforms
- **Targets & shared code** – The repo houses `Swiftfin` (iOS + iPadOS) and `Swiftfin tvOS` targets plus a `Shared/` tree for cross-platform components, coordinators, view models, services, and localized strings. The shared layer leans on `PlatformView` (`Shared/Objects/PlatformView.swift`) and `#if os(iOS)`/`#if os(tvOS)` conditionals to render per-platform UI.
- **Feature toggles** – `SwiftfinDefaults` centralizes app/user defaults and experimental flags. Downloads (and historically Live TV, Native Player, etc.) live under `Defaults.Experimental`, surfaced through `ExperimentalSettingsView` in each target.
- **Players** – Default playback uses Swiftfin (VLCKit) for codec breadth (HDR tone mapping, MKV/WebM, DTS, etc.) while the Native player enables PiP, framerate matching, and TLS 1.3 (see [players doc](Documentation/players.md)). Known limitations: audio delay on resume (issue #1762) and missing menus (#1761) when opting into Native on tvOS.
- **Library expectations** – [Documentation/libraries.md](Documentation/libraries.md) spells out that Shows/Movies/Music Videos/Mixed/Home Videos work today, Playlists and Music are blocked on future work, and Photos/Books are unsupported. Remind reporters of these boundaries before filing bugs.
- **Form factors** – iPad shares the iOS target but frequently uses `UIDevice.isPad` checks for orientation (`Swiftfin/App/SwiftfinApp.swift`). tvOS UI relies on focusable components (CollectionHStack/VGrid, `tvOSPicker`) and dedicated resources under `Swiftfin tvOS`.
- **Platform-specific settings** – tvOS exposes `.downActionShowsMenu` and `.confirmClose` defaults, while iOS leans on `PreferencesView` to host forms and uses `OverlayToastView` wrappers. When adding features, decide whether the code belongs under `Shared/` or per-target directories and gate APIs with `#if os(...)`.

## Environment, Dependencies, and Tooling
- **Toolchain** – Xcode 15+ (CI uses 16.4), Swift 6, Carthage for binary frameworks, Swift Package Manager for source dependencies, fastlane for CI/TestFlight, SwiftGen for codegen, SwiftFormat for style, Nuke for images, CoreStore for persistence, Factory/Defaults for DI & settings.
- **Carthage** – `Cartfile` pins VLCKit 3.5.0 for both iOS and tvOS. Run `carthage update --use-xcframeworks --platform ios,tvos` after cloning or when `Cartfile` updates.
- **Swift packages** – Key pins from `Package.resolved`:
  - `jellyfin-sdk-swift` 0.6.0 for API models/services.
  - UI helpers like `CollectionHStack`, `CollectionVGrid`, `BlurHashKit`, `SwiftUI-Introspect`, `TVOSPicker`.
  - Infrastructure libs `Factory`, `Defaults`, `PulseLogHandler`, `LNPopupController`.
  - `VLCUI` for embedding VLCKit controllers.
- **Automation** – `fastlane/Fastfile.swift` defines `buildLane` (local/CI builds) and `testFlightLane` (codesigned uploads). Secrets (certificates, App Store API keys) live in GitHub Actions; locally you can skip signing by passing `skipCodesigning: true`.
- **Logging & debugging** – `SwiftfinApp` wires `PersistentLogHandler` + `SwiftfinConsoleHandler` (DEBUG only) via Apple’s Logging API; logs live in `Shared/Logging`. Use `OverlayToastView` and `PreferencesView` when adding diagnostics UI.

## Examples of Good vs. Bad Contributions
- **Good bug report** – [#1764](https://github.com/jellyfin/Swiftfin/issues/1764) included step-by-step repro, device + OS matrix, server version, and a screen recording, enabling maintainers to confirm the duplicate and point to the TestFlight fix.
- **Needs-improvement bug report** – [#1815](https://github.com/jellyfin/Swiftfin/issues/1815) filed an issue for another fork (Streamyfin). Always verify the bundle identifier and UI before posting, and check for existing reports.
- **Good PR** – [`ErrorView` cleanup, #1798](https://github.com/jellyfin/Swiftfin/pull/1798) demonstrated the gold standard: extensive summary, UI videos for every platform, reuse of shared SwiftUI (PlatformView, Environment refresh), and TODOs for follow-up features.
- **Workflow PR** – [Version docs, #1802](https://github.com/jellyfin/Swiftfin/pull/1802) pushed policy content from a discussion into `Documentation/version.md`, updated the README badges, and iterated quickly on review feedback.
- **Needs-improvement PR** – [#1816](https://github.com/jellyfin/Swiftfin/pull/1816) fixed a tvOS flicker but sourced from the contributor’s `main` and lacked testing notes. Prefer a topic branch plus a short verification plan (“Tested on Apple TV 4K (tvOS 18.1), toggled between seasons in SeriesEpisodeSelector; flicker resolved”). That minimizes reviewer guesswork.

## Platform & Feature Guidance for Agents
- Before advising users to switch to the Native player, consult [players.md](Documentation/players.md) and issues #1761/#1762 to explain the missing chapter menu and subtitle limitations.
- Live TV, Downloads, and other incubating features appear behind Experimental toggles. After enabling them, a full app restart may be required (issue #1687). Communicate that clearly.
- When triaging playback problems, ask for:
  - Player type (Native vs Swiftfin),
  - TLS version and whether a proxy is used (see issue #1700),
  - Jellyfin server version and stream details (issue #1762 shows the expected level of detail).
- For multi-version episodes/movies, note the difference between server-side auto merge vs manual merges (issue #1763). Swiftfin currently mirrors the filename; improvements may require parsing identifiers client-side.

## Quickstart Checklist for New Agents
1. Clone the repo, install `carthage`, `swiftformat`, `swiftgen`, and run `carthage update --use-xcframeworks`.
2. Add `XcodeConfig/DevelopmentTeam.xcconfig` with your signing details.
3. Open `Swiftfin.xcodeproj`, select the appropriate scheme (`Swiftfin` for iOS/iPadOS, `Swiftfin tvOS` for Apple TV).
4. Run `swiftformat .` and `swiftgen config run` prior to committing.
5. Use `fastlane buildLane scheme:"Swiftfin"` (and tvOS) to match CI output when validating changes locally.
6. When documenting an issue/PR internally, link back to the relevant source (docs, issues, PRs) so future agents can retrace the decision.

---
This file is intended as a living companion for agents. Update it as new workflows emerge (e.g., additional experimental toggles, release process tweaks, or OS policy changes) and cross-link new documents so contributors can quickly orient themselves.
