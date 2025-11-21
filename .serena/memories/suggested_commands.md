# Suggested Commands
- `brew install carthage swiftformat swiftgen` – install the required tooling for dependency management, formatting, and code generation.
- `carthage update --use-xcframeworks` – download/refresh VLCKit binaries (run after cloning or Cartfile changes).
- `swiftformat .` (or `swiftformat . --lint --config .swiftformat`) – format or lint Swift sources; must pass before PRs.
- `swiftgen config run` – regenerate localized string constants after editing `Translations/` files.
- `fastlane buildLane scheme:"Swiftfin"` and `fastlane buildLane scheme:"Swiftfin tvOS"` – reproduce CI builds locally for iOS/iPadOS and tvOS.
- `fastlane testFlightLane keyID:… issuerID:… scheme:"Swiftfin" …` – used by CI/release automation to produce signed TestFlight builds when credentials are available.
- `xed Swiftfin.xcodeproj` or `open Swiftfin.xcodeproj` – launch the project in Xcode.
- Everyday repo work: `git status`, `git pull --rebase`, `rg <pattern>` for fast searches on macOS (Darwin environment).
