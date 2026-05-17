# Music Wallpaper — Ralph Loop Tasks

> Each task is sized to be completable in one ralph loop iteration (roughly 15–45 min of focused agent work). Tasks are ordered by dependency. Don't skip ahead — later tasks assume earlier ones are done.
>
> **Workflow per task:** read the task → read any files it touches → make the change → run tests/build → tick the box → write the one-line "what I learned" note.
>
> Status legend: `[ ]` todo · `[~]` in progress · `[x]` done · `[!]` blocked (add reason).

---

## Phase 0 — Tooling & Project Setup

> Tasks 0.1–0.4 are **one-time-ever setup on your Mac** — once done, you never repeat them, even for future projects. Tasks 0.5 onward are this project's bootstrap.
>
> **Your identity for Git (used in 0.2):** name `Cyrus`, email `cy102611@gmail.com`.
> **GitHub repo URL (fill in after task 0.3):** `https://github.com/<cy102611>/MusicWallpaper.git`

### One-time Git + GitHub setup (do once, ever)

- [ ] **0.1** Open Terminal (⌘+Space, type "Terminal") and run `git --version`. If you see a version string, move on. If macOS prompts to install Command Line Tools, click Install and wait for it to finish (a few minutes).
- [ ] **0.2** Configure Git globally with your identity. Copy each line into Terminal and press Enter:
  ```
  git config --global user.name "Cyrus"
  git config --global user.email "cy102611@gmail.com"
  git config --global init.defaultBranch main
  ```
  Verify with `git config --global --list` — you should see all three values echoed back.
- [ ] **0.3** Create a GitHub account (or sign in) at https://github.com using `cy102611@gmail.com`. On github.com, click the green **New** button to create a private repository named `MusicWallpaper`. **Do NOT check** any of the "Initialize this repository with…" boxes (no README, no .gitignore, no license — you already have them locally). After creation, copy the HTTPS clone URL shown on the empty repo page — it looks like `https://github.com/<your-github-username>/MusicWallpaper.git`. Paste that URL into the placeholder at the top of this file. You'll need it again in task 0.9.
- [ ] **0.4** Authenticate Git with GitHub once. Easiest path: download [GitHub Desktop](https://desktop.github.com), launch it, sign in with your GitHub account, then quit. That single sign-in caches your credentials in macOS Keychain so command-line `git push` works forever after — you do not need to use Desktop for anything else.

### Project bootstrap (per-project, do once for this project)

- [ ] **0.5** Create new Xcode project. App name `MusicWallpaper`, bundle id `com.cyrus.musicwallpaper`, interface SwiftUI, language Swift, min deployment iOS 18.0. Save the project to a stable location (e.g. `~/Documents/Projects/MusicWallpaper`). Quit Xcode when done.
- [ ] **0.6** In Terminal, `cd` into the project root (the folder containing the `.xcodeproj` file — the easy way: type `cd ` then drag the folder from Finder into Terminal and press Enter). Run `git init`. You should see "Initialized empty Git repository in …/.git/".
- [ ] **0.7** Create a `.gitignore` file at the project root containing exactly:
  ```
  # macOS
  .DS_Store

  # Xcode user state
  xcuserdata/
  *.xcuserstate

  # Secrets — never commit
  Secrets.xcconfig

  # Build artefacts
  build/
  DerivedData/
  ```
- [ ] **0.8** Make the first commit:
  ```
  git add .
  git commit -m "chore: initial commit, empty Xcode project"
  ```
- [ ] **0.9** Hook up the GitHub remote and push. Replace `<your-github-username>` with your actual GitHub username from task 0.3:
  ```
  git remote add origin https://github.com/<your-github-username>/MusicWallpaper.git
  git branch -M main
  git push -u origin main
  ```
  Refresh github.com in your browser — you should now see your project files. From here on, every task's checkbox flip should be paired with a `git commit -m "<scope>: <summary>"` and (whenever you want a backup) a `git push`.
- [ ] **0.10** Enable Swift 6 language mode and strict concurrency checking (Complete) in build settings. Build — fix any warnings the template generates. Commit: `chore: enable swift 6 strict concurrency`.
- [ ] **0.11** Create folder structure in the Xcode project: `App/`, `Features/`, `Services/`, `Models/`, `Utilities/`, `Resources/`. Each is a real folder on disk (not just a group). Commit: `chore: add feature folder structure`.
- [ ] **0.12** Add an `App Group` capability (`group.com.cyrus.musicwallpaper`) and enable Background Modes (Audio, Background fetch, Background processing). Add Photos add-only usage description and Apple Music usage description to Info.plist. Commit: `chore: add capabilities and usage descriptions`.
- [ ] **0.13** Create `Secrets.xcconfig` with placeholder keys `LASTFM_API_KEY`, `GOOGLE_CSE_KEY`, `GOOGLE_CSE_CX`. Wire it up via build configuration. Surface the keys to runtime via Info.plist entries that read `$(LASTFM_API_KEY)` etc. Commit a `Secrets.xcconfig.example` with placeholder values only — the real `Secrets.xcconfig` is gitignored by task 0.7. Commit: `chore: add secrets scaffolding`.
- [ ] **0.14** Add a `Logger+App.swift` extension that pre-configures subsystems (`app`, `monitor`, `resolver`, `cache`, `wallpaper`, `intents`). Wire one log line in the app entrypoint so you can see it in Console.app. Commit: `feat: add OSLog subsystems`.

## Phase 1 — Models & Errors

- [ ] **1.1** Create `Models/NowPlayingTrack.swift` — a `struct` with `trackID`, `title`, `artist`, `albumTitle`, `albumID`, `artistID`, `duration`, `artworkURL`. `Sendable`, `Equatable`, `Hashable`.
- [ ] **1.2** Create `Models/ResolvedArtwork.swift` — a `struct` with `image: UIImage`, `source: ArtworkSource` enum, `originalURL: URL?`, `resolvedAt: Date`. `Sendable`.
- [ ] **1.3** Create `Models/WallpaperError.swift` — `enum WallpaperError: Error, LocalizedError` with cases: `notAuthorized(Permission)`, `noTrackPlaying`, `providerFailed(source: ArtworkSource, underlying: Error?)`, `imageTooSmall`, `photosWriteFailed`, `cacheCorrupt`. Add `errorDescription` for each.
- [ ] **1.4** Add unit test target. Write a single passing test that asserts `WallpaperError.noTrackPlaying.errorDescription` is non-nil. Confirm the test runner works.

## Phase 2 — Logging & Utilities

- [ ] **2.1** Implement `Utilities/AsyncRetry.swift` — generic `func retry<T>(times: Int, backoff: Duration, _ op: () async throws -> T) async throws -> T`. Exponential backoff. Cancellation-safe.
- [ ] **2.2** Implement `Utilities/Hashing.swift` — a stable `sha256(_:)` over a tuple of strings, returning hex. Used as cache key.
- [ ] **2.3** Implement `Utilities/Timeout.swift` — `func withTimeout<T: Sendable>(_ duration: Duration, operation: @Sendable () async throws -> T) async throws -> T`. Built on `TaskGroup` with one race-loser. Throws `WallpaperError.providerTimedOut` on expiry. Cancellation-safe both ways.
- [ ] **2.4** Add `WallpaperError.providerTimedOut(source: ArtworkSource)` case (extend the enum from 1.3) and `WallpaperError.smtpFailed(reason: String)` for later use.
- [ ] **2.5** Add tests for all three utilities. Cover happy path, cancellation, retry eventual failure, and timeout firing within ±50 ms of the budget.

## Phase 3 — Image Cache

- [ ] **3.1** Create `Services/ImageCache.swift` as an `actor`. In-memory `NSCache<NSString, UIImage>` with 60 MB cost limit. Methods: `image(forKey:) async -> UIImage?`, `store(_:forKey:cost:) async`.
- [ ] **3.2** Add disk tier — `Application Support/WallpaperCache/<key>.heic`. Lazy LRU based on file `contentModificationDate`. Methods: `imageFromDisk(forKey:) async -> UIImage?`, `persist(_:forKey:) async throws`.
- [ ] **3.3** Add eviction: when disk total > 500 MB, delete oldest files until under 400 MB (high/low watermark). Run on every persist.
- [ ] **3.4** Write tests with a temp directory. Cover: store→retrieve, memory hit, disk hit, eviction triggers correctly.

## Phase 4 — Artwork Providers

- [ ] **4.1** Define `Services/Providers/ArtworkProviding.swift` — `protocol ArtworkProviding: Sendable { var source: ArtworkSource { get }; func fetchArtwork(for track: NowPlayingTrack) async throws -> UIImage? }`.
- [ ] **4.2** Implement `AppleMusicArtworkProvider` using MusicKit `Song.artwork?.url(width:height:)`. Sizes the request to the device's native resolution.
- [ ] **4.3** Implement `UserOverrideStore` — looks for `<App Group>/Overrides/album-<albumID>.jpg`, falls back to `artist-<artistID>.jpg`. Provides also a `saveOverride(image:for:scope:)` helper for future UI.
- [ ] **4.4** Implement `LastFmProvider`. Hits `album.getInfo` first, falls through to `artist.getInfo`. Returns the largest `<image>` URL ("mega" / "extralarge"). Wraps every network call in `withTimeout(.seconds(timeoutBudget))` where `timeoutBudget` is read from `UserDefaults.networkTimeoutSeconds` (default 30). Uses `AsyncRetry` *inside* the timeout, not outside, so total wall-time is bounded.
- [ ] **4.5** Implement `GoogleCustomSearchProvider`. Query string: `"<artist> <album> album cover"` for albums, `"<artist> portrait"` for artists. Filters out images smaller than device width / 2. Same timeout wrapping as 4.4.
- [ ] **4.6** Implement `FallbackGradientProvider` — deterministic gradient from a hash of the track ID, rendered to a `UIImage` at device resolution. Always succeeds.
- [ ] **4.7** Per-provider tests with recorded JSON fixtures (no live network in tests). Add fixtures under `Tests/Fixtures/`.

## Phase 5 — Resolver Pipeline

- [ ] **5.1** Create `Services/ResolverPipeline.swift` as an `actor`. Init takes a `providerRegistry: [ArtworkSource: any ArtworkProviding]` (map, not array — order is decided at resolve time) and the `ImageCache`.
- [ ] **5.2** Create `Services/SourcePreferences.swift` — wraps `UserDefaults` keys `sourceOrder` (array of `ArtworkSource` raw values) and `disabledSources` (set of raw values). Provides a default order matching §4 of `OVERVIEW.md`. Notifies subscribers on change via `AsyncStream`.
- [ ] **5.3** Implement `resolve(for: NowPlayingTrack) async -> ResolvedArtwork` — check cache, then iterate providers in `SourcePreferences.currentOrder()`, skipping disabled. First non-nil image wins, persists to cache, returns. Always returns something thanks to `FallbackGradientProvider` (which can't be disabled).
- [ ] **5.4** Add cancellation: the method is `async` and respects `Task.checkCancellation()` between providers. Document that callers should wrap in a task tied to the current track.
- [ ] **5.5** Add image post-processing — center-crop to device aspect ratio at native scale, encode HEIC. Reject sources smaller than `min(deviceWidth/2, 600)` and fall through. Embed `IPTC:Keywords = [artistName]` and `EXIF:UserComment = "Music Wallpaper / <providerName>"` using `CGImageDestination`.
- [ ] **5.6** Integration test that proves the chain: feed it a fake provider list where each says "no" until the 3rd, assert source matches the 3rd. Also assert that disabling source #3 in `SourcePreferences` makes the chain skip to #4.
- [ ] **5.7** Integration test for timeout: feed it a fake `LastFmProvider` that sleeps 60 s with a 5 s budget. Assert it returns via the next provider, not by hanging.

## Phase 6 — Now Playing Monitor

- [ ] **6.1** Create `Services/NowPlayingMonitor.swift`, a `@MainActor` `@Observable` class. Subscribes to `MPMusicPlayerController.systemMusicPlayer` notifications for track change and playback state.
- [ ] **6.2** Bridge to MusicKit: when track changes, look up the corresponding `Song` via `MusicCatalogResourceRequest` to enrich metadata (high-res artwork URL).
- [ ] **6.3** Expose `currentTrack: NowPlayingTrack?` and `upcomingTracks: [NowPlayingTrack]` (next 3) as observable properties. Also expose an `AsyncStream<NowPlayingTrack>` of track changes.
- [ ] **6.4** Add a "settle" debounce: only emit a track change after 400 ms of stability, so rapid skipping doesn't trigger a wallpaper update per skip.
- [ ] **6.5** Tests with a fake player. Cover: track change emits, pause does not emit, rapid skip emits only the settled track.

## Phase 7 — Output Channels (Photos, Shared Container, Email)

- [ ] **7.1** Create `Services/OutputChannels/OutputChannel.swift` — `protocol OutputChannel: Sendable { var id: ChannelID { get }; var isEnabled: Bool { get async }; func emit(_ artwork: ResolvedArtwork, track: NowPlayingTrack) async throws }`. Define `enum ChannelID: String { case photosAlbum, sharedContainer, emailTrigger }`.
- [ ] **7.2** Create `Services/OutputChannels/ChannelPreferences.swift` — `UserDefaults`-backed enable/disable per channel. `sharedContainer` is always enabled (hardcoded). Exposes an `AsyncStream` for live changes.
- [ ] **7.3** Implement `SharedContainerChannel`. Writes `current.png` and `current.json` (artist, album, track, source, resolvedAt) to the App Group container atomically (write to temp, then rename).
- [ ] **7.4** Implement `PhotosAlbumChannel`. On first emit, creates the `"Music Wallpaper"` Photos album. For each emit: lazily ensure a sub-album named after the artist exists, insert the image into both, set `IPTC:Keywords = [artistName]` in the image data passed to `PHAssetCreationRequest`. Only requests `.addOnly` Photos auth when this channel is first enabled.
- [ ] **7.5** Photos cleanup: keep at most 50 assets per artist sub-album and 200 in the parent album. Track our own assets via SwiftData `WallpaperAsset { localIdentifier, artistID, createdAt }`. Delete oldest beyond the cap. Never touch assets we didn't create.
- [ ] **7.6** Create `Services/OutputChannels/SMTP/Keychain.swift` — thin wrapper for storing/reading `SMTPCredentials { host, port, username, password, useTLS, fromAddress, toAddress }`. Account: `com.cyrus.musicwallpaper.smtp`. Accessibility: `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`.
- [ ] **7.7** Create `Services/OutputChannels/SMTP/SMTPClient.swift` — minimal SMTP client built on `Network.framework` `NWConnection` with STARTTLS / implicit TLS. Supports AUTH LOGIN and AUTH PLAIN. Sends a single MIME multipart message with one JPEG attachment. No external dependencies. Methods: `connect() async throws`, `send(_ message: SMTPMessage) async throws`, `disconnect() async`.
- [ ] **7.8** Implement `EmailTriggerChannel`. Composes subject `MusicWallpaper :: <artist>`, body = JSON manifest with track metadata, attachment = JPEG (quality 0.9, capped at 4 MB by re-encoding if needed). Uses `SMTPClient`. Reads credentials from Keychain. If credentials missing or invalid, raises a one-shot user notification ("Email channel disabled — open app to re-test") and self-disables in `ChannelPreferences`.
- [ ] **7.9** Email throttling: an `actor EmailRateLimiter` enforces "at most one email per 60 s per artist." If a new track arrives during cooldown, store it as the pending value, debouncing — when cooldown expires, send for the most recent track only (or skip if it changed back to the original).
- [ ] **7.10** Create `Services/WallpaperOut.swift` — the coordinator. Owns the three channel instances. `emit(_ artwork:, track:)` calls `emit` on every enabled channel concurrently in a `withTaskGroup`, logs per-channel success/failure, never throws (channel failures are logged, not propagated, so one bad channel can't break the others).
- [ ] **7.11** Channel tests: mocked `PHPhotoLibrary` wrapper protocol for Photos channel (album creation, sub-album per artist, IPTC keywords present, cleanup threshold). FileManager-based assertions for SharedContainer. For Email: a fake SMTP server fixture (use `swift-nio` only inside the test target if needed, otherwise a localhost listener) confirms a well-formed MIME message arrives. Throttling test asserts exactly one send across 10 rapid emits.
- [ ] **7.12** "Send test email" helper: a public `EmailTriggerChannel.sendTestEmail() async throws` that bypasses the rate limiter and sends a fixed payload. Wired up to the Settings UI in Phase 11.

## Phase 8 — Prefetch

- [ ] **8.1** In `ResolverPipeline`, add `prefetch(tracks: [NowPlayingTrack]) async` — runs the resolver with `writeToWallpaper = false`, only populates cache, max 3 concurrent.
- [ ] **8.2** In `NowPlayingMonitor`, on every track change, schedule `Task.detached(priority: .utility)` that calls `prefetch(tracks: upcomingTracks)`. Store the task; cancel if track changes again.
- [ ] **8.3** Add a feature flag (`UserDefaults.prefetchEnabled`, default true) so it can be turned off for testing.
- [ ] **8.4** Test that prefetch never writes to Photos and respects cancellation when the queue changes.

## Phase 9 — App Intents (Shortcuts surface)

- [ ] **9.1** Create `Features/Intents/SetWallpaperForCurrentTrackIntent.swift`. `AppIntent` that resolves the current track, runs the pipeline, writes to Photos + shared container, returns the file URL as `IntentFile`.
- [ ] **9.2** Create `RefreshCurrentArtworkIntent` — same as 9.1 but bypasses cache (force re-fetch).
- [ ] **9.3** Create `PrefetchUpcomingQueueIntent` — runs prefetch for the next 3 tracks. Useful for user-built shortcuts.
- [ ] **9.4** Add an `AppShortcutsProvider` with discoverable phrases ("Update my music wallpaper").
- [ ] **9.5** Manual test on device: open Shortcuts app, confirm the three intents appear and run.

## Phase 10 — Background Tasks

- [ ] **10.1** Register a `BGAppRefreshTask` (`com.cyrus.musicwallpaper.refresh`) that wakes the `NowPlayingMonitor` and runs `SetWallpaperForCurrentTrackIntent` if music is playing.
- [ ] **10.2** Register a `BGProcessingTask` (`com.cyrus.musicwallpaper.prefetch`) that runs prefetch when on Wi-Fi + charging.
- [ ] **10.3** Schedule reschedules at the end of each task execution. Add `Info.plist` `BGTaskSchedulerPermittedIdentifiers` entries.
- [ ] **10.4** Manual test using `e -l objc -- (void)[[BGTaskScheduler sharedScheduler] _simulateLaunchForTaskWithIdentifier:@"…"]` in lldb. Confirm both tasks run and complete.

## Phase 11 — SwiftUI Shell

- [ ] **11.1** Create `App/MusicWallpaperApp.swift` (SwiftUI `@main`). Inject services into the environment via `.environment(...)`. Use the new `@Observable` flow, not `ObservableObject`.
- [ ] **11.2** Build `Features/Onboarding/OnboardingView.swift` — four pages: welcome → permissions (one button per permission, shows current status) → output channel picker (Photos toggle, Email toggle with "configure SMTP" link) → Shortcuts setup (deep links for both the file-trigger and email-trigger shortcut templates + brief instructions).
- [ ] **11.3** Build `Features/Status/StatusView.swift` — shows currently playing track, currently set wallpaper image, cache stats (count, size), per-channel last-emit status, last error if any. Buttons: "Refresh now", "Open Shortcuts", "Open Settings".
- [ ] **11.4** Build `Features/Settings/SettingsView.swift` — entry point. Sections: Image Sources, Output Channels, Timeouts, About.
- [ ] **11.5** Build `Features/Settings/SourcePriorityView.swift` — a SwiftUI `List` with `.onMove` enabled, drag handles, and a toggle per row to disable the source. Cache layers and Fallback Gradient are rendered as locked rows (greyed handle, no toggle). Writes through to `SourcePreferences` on every change.
- [ ] **11.6** Build `Features/Settings/ChannelTogglesView.swift` — three rows: Photos Album (toggle), Shared Container File (locked on, info icon explains why), Email Trigger (toggle, expands to show "Configure SMTP" navigation link when on).
- [ ] **11.7** Build `Features/Settings/SMTPSetupView.swift` — form for host / port / username / password (`SecureField`) / from / to / TLS toggle. "Send test email" button. On success, save to Keychain via `Services/OutputChannels/SMTP/Keychain.swift` and enable the channel. On failure, show the SMTP server's response verbatim under the button (it's almost always the most useful error).
- [ ] **11.8** Build `Features/Settings/TimeoutsView.swift` — a `Slider` for "Network provider timeout" bound to `UserDefaults.networkTimeoutSeconds` (range 5–60, default 30, step 5). Show the current value as a label. Brief explanatory text under it.
- [ ] **11.9** Hook the onboarding flow to a `UserDefaults` `hasCompletedOnboarding` flag. First-launch shows onboarding; subsequent launches go straight to StatusView. Settings is always reachable via a gear icon on StatusView.
- [ ] **11.10** Polish: dark/light mode, dynamic type up to XXL, VoiceOver labels on all interactive elements (especially the drag-reorder list — VoiceOver should announce "moved Last.fm album to position 4 of 7"). Run the Accessibility Inspector audit and fix anything flagged.

## Phase 12 — Shortcut Template

- [ ] **12.1** Build Shortcut Template A — **File Trigger**: Run `SetWallpaperForCurrentTrackIntent` → Get File from shared App Group `current.png` → **Set Wallpaper Photo**. Save as "Music Wallpaper — From File."
- [ ] **12.2** Build Shortcut Template B — **Email Trigger**: Personal Automation, trigger = "When I receive an email", filter sender = the user's own configured sender, subject contains "MusicWallpaper" → Get Attachments → **Set Wallpaper Photo** with the first attachment. Save as "Music Wallpaper — From Email."
- [ ] **12.3** Build an optional Personal Automation: "When Apple Music is opened" → run Template A. (For users who don't want the email channel.) Document both flows in `OVERVIEW.md`'s onboarding section.
- [ ] **12.4** Export both shortcuts as iCloud links. Bake them into the onboarding view as two side-by-side "Install" buttons labeled "File Trigger" and "Email Trigger." The onboarding view greys out the email button until SMTP is configured.

## Phase 13 — Hardening

- [ ] **13.1** Memory profile: run a 30-minute session with Instruments (Allocations + Leaks). Fix anything above the 80 MB ceiling. No leaks allowed.
- [ ] **13.2** Cold-start profile: launch time must be under 600 ms on iPhone 15. Trim anything synchronous on startup.
- [ ] **13.3** Network profile: confirm Last.fm and Google calls are only made when prior providers miss. Add an analytics counter (local OSLog) for per-provider hit rates and per-channel emit success rates.
- [ ] **13.4** SMTP hardening: confirm `SMTPClient` uses TLS 1.2 minimum, validates the server certificate (no `allowAnyServerCertificate`), refuses to send if the connection downgrades to plaintext mid-handshake. Confirm Keychain items are *not* synced to iCloud Keychain.
- [ ] **13.5** Add the Privacy Manifest (`PrivacyInfo.xcprivacy`) declaring required reason API usage (UserDefaults, FileManager, SystemBootTime) and listing the third-party domains (Last.fm, Google CSE, and an `NSAllowsArbitraryLoadsInWebContent`-style note about the user's SMTP host being unknown at compile time).
- [ ] **13.6** Confirm "self-disable" behavior: kill the network mid-SMTP send; confirm the email channel reports failure and self-disables after 3 consecutive failures, with a non-modal notification to re-test in Settings.
- [ ] **13.7** Run the full edge-case checklist from `OVERVIEW.md` §8.2. Fix anything that fails. Re-run.

## Phase 14 — Release Prep

- [ ] **14.1** App icon (1024×1024) + all required sizes. Use SF Symbol `music.note.tv` as inspiration if you don't have art yet — placeholder is fine for TestFlight.
- [ ] **14.2** Screenshots for App Store (iPhone 6.9", 6.5", 5.5"). Three screens: Now Playing, Wallpaper Preview, Onboarding.
- [ ] **14.3** Write the App Store description making clear that wallpaper changes are routed through Shortcuts (Apple is strict about this — be honest).
- [ ] **14.4** TestFlight build. Distribute to yourself. Run the full §8 checklist on the TestFlight build (not Debug).
- [ ] **14.5** Submit for review. Be ready to explain the Shortcuts hand-off to a reviewer.

---

## Backlog (post-v1, not in scope yet)

- [ ] Lyric-driven wallpaper using Apple Music synced lyrics.
- [ ] User-curated "preferred image" for an artist via swipe-up gesture.
- [ ] Spotify support via SpotifyWebAPI (separate auth flow).
- [ ] Live Photo wallpapers for songs with music videos.
- [ ] Watch app showing the current source provider.

---

## Notes from the Loop

> Each completed task should leave a one-liner here, dated. Keeps a running history of surprises so the next pass knows what to watch for.

- 2026-MM-DD — *<first note will go here>*
