# Task Completion Checklist
1. **Ensure clean tree** – run `git status` to verify only intentional changes remain; remove stray artifacts.
2. **Format & lint** – execute `swiftformat .` (or `swiftformat . --lint --config .swiftformat`) so CI’s lint job passes.
3. **Regenerate assets** – if you touched strings or SwiftGen inputs, run `swiftgen config run` so generated files are updated.
4. **Build both targets** – run `fastlane buildLane scheme:"Swiftfin"` and `fastlane buildLane scheme:"Swiftfin tvOS"` (or build via Xcode) to ensure iOS/iPadOS and tvOS compile.
5. **Add testing notes** – document which device/OS/player combinations you exercised, plus any screenshots/videos for UI work.
6. **Reference issues & labels** – update PR description with `Closes #…`, apply labels (`bug`, `enhancement`, platform), and link relevant documentation if applicable.
7. **Review docs** – when introducing behavior changes, update `Documentation/` or README references so the guidance stays current.
