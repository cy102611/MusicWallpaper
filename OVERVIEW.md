# Music Wallpaper — App Overview & Engineering Guide

> A zero-touch iOS app that swaps your iPhone wallpaper to match whatever Apple Music is currently playing — album art, artist photo, or a hand-picked image. Built natively in SwiftUI.

---

## 1. Product Vision

**One sentence:** When the song changes, the wallpaper changes — automatically, with no taps.

**Core promise:** The user opens the app *once* to grant permissions and run a one-time Shortcuts setup. After that, the app is invisible. They press play in Apple Music and their Lock Screen / Home Screen quietly follows along.

**Out of scope (on purpose):**
- No social features, no sharing, no accounts.
- No manual song-by-song picking. The app decides; the user lives with it.
- Apple Music only.

---

## 2. Critical Platform Reality (Read This First)

iOS **does not** let third-party apps set the wallpaper directly. There is no public API. Any plan that ignores this fails. The workable path is:

1. The app **detects** the now-playing track via `MusicKit` / `MPNowPlayingInfoCenter`.
2. The app **resolves an image** for that track (priority chain: Apple Music artwork → user-saved override → Last.fm → Google Custom Search → fallback color).
3. The app **writes the image** to a dedicated Photos album *and* a known shared container path.
4. The app **exposes an App Intent** (`SetWallpaperForCurrentTrackIntent`) for the Shortcuts app.
5. The **user's one-time Shortcuts automation** ("When Apple Music starts playing" or a Personal Automation on app open) runs that intent. Where iOS still permits a "Set Wallpaper" action (via Focus Filters or legacy shortcut on supported OS versions) it is used; otherwise the image is staged for the next Focus Mode wallpaper rotation.

The app's job is to do **everything Apple lets it do**, then hand the final image to Shortcuts/Focus on a silver platter. Document this trade-off clearly in the README so reviewers don't reject for "false advertising."

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   SwiftUI App Shell                     │
│  (Onboarding view, Permission view, Status dashboard)   │
└────────────────────┬────────────────────────────────────┘
                     │
       ┌─────────────┴─────────────┐
       │                           │
┌──────▼──────────┐        ┌───────▼────────┐
│ NowPlayingMonitor│        │ AppIntentLayer │
│ (MusicKit +      │        │ (Shortcuts &   │
│  MPNowPlaying)   │        │  Focus hooks)  │
└──────┬──────────┘        └───────┬────────┘
       │                           │
       └─────────────┬─────────────┘
                     │
              ┌──────▼───────┐
              │  Resolver    │  ← Decides which image to use
              │  Pipeline    │     (priority chain below)
              └──────┬───────┘
                     │
       ┌─────────────┼─────────────┬──────────────┐
       │             │             │              │
┌──────▼────┐ ┌──────▼────┐ ┌─────▼─────┐ ┌──────▼─────┐
│ AppleMusic│ │ UserOverr │ │ LastFm    │ │ GoogleCSE  │
│ Artwork   │ │ ide Store │ │ Provider  │ │ Provider   │
└──────┬────┘ └──────┬────┘ └─────┬─────┘ └──────┬─────┘
       └─────────────┴─────────────┴──────────────┘
                     │
              ┌──────▼───────┐
              │ ImageCache   │  ← NSCache + disk LRU
              └──────┬───────┘
                     │
              ┌──────▼───────┐
              │ WallpaperOut │  ← Fans out to enabled channels
              └──────┬───────┘
                     │
       ┌─────────────┼─────────────────────┐
       │             │                     │
┌──────▼──────┐ ┌────▼───────┐ ┌───────────▼────────┐
│ PhotosAlbum │ │ Shared FM  │ │ EmailTrigger       │
│ (optional)  │ │ Container  │ │ (SMTP → Shortcuts) │
└─────────────┘ └────────────┘ └────────────────────┘
```

### Module responsibilities

- **NowPlayingMonitor** — subscribes to `MPMusicPlayerController.systemMusicPlayer` notifications and `MusicKit.ApplicationMusicPlayer.shared.state`. Emits a Combine `AnyPublisher<NowPlayingTrack, Never>`. Also computes the *next* track in the queue so the resolver can prefetch.
- **Resolver Pipeline** — async/await sequence that asks each provider in order. First non-nil wins. Cached results short-circuit network calls.
- **Providers**
  - `AppleMusicArtworkProvider` — uses `Song.artwork?.url(width:height:)` from MusicKit (no network call beyond Apple's CDN; usually instant).
  - `UserOverrideStore` — looks up a per-album / per-artist image the user dropped into the in-app folder. Backed by `FileManager` in App Group container.
  - `LastFmProvider` — calls `album.getInfo` then `artist.getInfo`. Requires API key in `Secrets.xcconfig`.
  - `GoogleCustomSearchProvider` — last resort for artists/albums with no good art elsewhere. Requires CSE key + cx.
- **ImageCache** — two-tier: `NSCache<NSString, UIImage>` in memory (capacity 60 MB), and an on-disk LRU under `Application Support/WallpaperCache/` (cap 500 MB, evicted oldest-first). Keyed by a stable hash of `(artistID, albumID, trackID, providerName, dimensions)`.
- **WallpaperOut** — fans the resolved image out to whichever **output channels** the user has enabled (see §6). Channels are independent: any combination of Photos album, shared-container file, and email trigger can be on. Every emitted image carries the artist name as a tag (Photos album sub-album + EXIF `IPTC:Keywords` + email subject prefix), so the user can later filter by artist anywhere they look.
- **AppIntentLayer** — exposes `SetWallpaperForCurrentTrackIntent`, `PrefetchUpcomingQueueIntent`, and `RefreshCurrentArtworkIntent`. These are the surfaces a user's automation calls into.
- **Background work** — `BGProcessingTask` for opportunistic prefetch when the app is backgrounded but the queue is known. Also a periodic `BGAppRefreshTask` to keep the now-playing observer warm.

---

## 4. Image Resolution Priority (the Decision Engine)

When a track starts, the resolver runs this chain. **First hit wins.** The pipeline is async and cancellable — if the user skips the song mid-fetch, the in-flight request is cancelled.

**The order is user-configurable.** What you see below is the default. In Settings the user can drag sources up and down, and toggle any *network* source (UserOverride, Last.fm album, Last.fm artist, Google CSE) off entirely. Cache layers and the fallback gradient are always on and can't be reordered (they bookend the chain). The configured order is persisted via `UserDefaults` and re-read on every track change so changes apply immediately.

| Order | Source                      | Reorderable? | When it wins                                                   |
|-------|-----------------------------|--------------|----------------------------------------------------------------|
| 1     | In-memory cache             | No (always first) | Same track played recently in this session                |
| 2     | Disk cache                  | No (always second)| Same track played at least once before                    |
| 3     | UserOverrideStore           | Yes          | User dropped a custom image for this album/artist              |
| 4     | Apple Music artwork (album) | Yes          | Default path — works for ~95% of streamed catalog              |
| 5     | Last.fm album image         | Yes          | Apple Music returns generic / missing artwork                  |
| 6     | Last.fm artist image        | Yes          | Album lookup empty (rare bootlegs, classical, etc.)            |
| 7     | Google CSE (artist photo)   | Yes          | Everything else failed                                          |
| 8     | Fallback gradient           | No (always last) | All network sources failed — render a deterministic gradient |

**Per-provider hard timeout.** Network providers (Last.fm and Google CSE) get **30 seconds** before the resolver gives up on them and falls through to the next source. The timeout is enforced with `withTimeout(_:operation:)` (a small helper around `TaskGroup` cancellation) — not just trusting `URLSession.timeoutIntervalForRequest`, because TLS handshakes and DNS can blow past that quietly. The 30 s budget is also user-configurable in Settings (range 5–60 s, default 30). Apple Music artwork and UserOverride aren't bounded by this — they're effectively local.

Every chosen image is sized to **the device's native screen scale × 2** (e.g. 1290×2796 on iPhone 15 Pro Max) and cropped center-weighted before being written out. EXIF metadata is rewritten so `IPTC:Keywords` contains the artist's name and `EXIF:UserComment` contains the source provider — useful for debugging and for the user's own organization.

---

## 4.5 Output Channels (the Hand-Off)

Once the resolver returns a `ResolvedArtwork`, `WallpaperOut` fans it out to the channels the user enabled. Channels are independent — any combination can be on, including none (in which case the app is still useful as a now-playing image cache). Each channel is its own protocol-fronted service so it can be tested in isolation.

**Channel A — Photos Album** *(optional, default on)*
- Inserts the image into a top-level `"Music Wallpaper"` Photos album, with one sub-album per artist (e.g. `"Music Wallpaper / Radiohead"`). Sub-albums are created lazily.
- Image metadata carries `IPTC:Keywords = <artist name>` so Photos's search field finds it.
- Rolling cleanup: keep at most 50 images per artist sub-album and 200 total in the parent. Older images created by this app are deleted, tracked by `localIdentifier` in SwiftData.
- Requires `PHPhotoLibrary` add-only authorization.

**Channel B — Shared Container File** *(always on; cheap and required for any Shortcut)*
- Writes the latest image as `current.png` plus a `current.json` with `{ artist, album, track, source, resolvedAt }` to the App Group's shared container.
- Any Shortcut the user builds can read this file directly — it's the simplest possible hand-off.

**Channel C — Email Trigger** *(optional, off by default — user opts in during onboarding)*
- Purpose: drive a Shortcuts Personal Automation whose trigger is **"When I receive an email."** Apple's *Set Wallpaper Photo* action is available inside Shortcuts; the missing piece is a reliable way to wake a Shortcut on track change from a backgrounded third-party app. Email is the cleanest stable trigger Apple gives us.
- The app composes an email *to the user themselves* with:
  - **From**: an account the user configured in onboarding (their own SMTP).
  - **To**: the same address.
  - **Subject**: `MusicWallpaper :: <artist>` — the artist name is the tag, so the Shortcut's email filter can also be per-artist if the user wants.
  - **Body**: a small JSON manifest with track metadata.
  - **Attachment**: the resolved image as a JPEG (capped at 4 MB to fit through every provider).
- Transport: the app sends via SMTP using credentials the user enters once in onboarding. Credentials live in Keychain (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`). No third-party email service is required, no Anthropic / app-author server is in the loop — the user's email talks directly to the user's email.
- Throttling: at most one email per 60 seconds (per artist) to avoid Gmail / iCloud rate-limit hells. If a new track arrives during the cooldown, the most recent track wins and the email is sent after cooldown expires.
- The user's matching Shortcuts automation: **"When I receive an email"** with sender = your own address and subject contains `MusicWallpaper` → **Get Attachments from Input** → **Set Wallpaper Photo**. A pre-built template shortcut is shipped via an iCloud link in onboarding (see §12).

**User-facing toggle.** Settings has a "Where should wallpapers go?" panel with three switches (Photos album, Shared container — read-only on, Email trigger). Email shows a sub-panel for SMTP config (host, port, username, password — masked) and a "Send test email" button. Nothing autosends until the test succeeds.

---

## 5. Prefetch Strategy

If on-demand fetch takes > 800 ms (rough budget for "feels instant"), the resolver kicks off prefetch for the **next 3 queued tracks**. Implementation:

1. `NowPlayingMonitor.upcomingTracks` exposes the next N tracks from `ApplicationMusicPlayer.shared.queue`.
2. On every track change, schedule a background `Task.detached(priority: .utility)` that runs the resolver pipeline for tracks 2, 3, 4 — populating disk cache only (no wallpaper write).
3. Throttle: never more than 3 concurrent prefetches. Cancel all if user changes queue or pauses.

---

## 6. Tech Stack & Frameworks

| Concern                  | Framework / Tool                                    |
|--------------------------|-----------------------------------------------------|
| UI                       | SwiftUI (iOS 18 SDK, Swift 6)                       |
| Music observation        | MusicKit, MediaPlayer                               |
| Concurrency              | Swift Concurrency (async/await, actors, AsyncStream)|
| Reactive glue            | Combine (only where AsyncStream is awkward)         |
| Background work          | BackgroundTasks (BGProcessingTask, BGAppRefreshTask)|
| Photos write             | Photos / PhotosUI                                   |
| Shortcuts surface        | App Intents                                         |
| Networking               | URLSession + async/await                            |
| Email transport          | Raw SMTP over `Network.framework` NWConnection + TLS (no third-party SMTP lib for v1) |
| Secure credential store  | Keychain Services (`kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`) |
| Image handling           | UIKit (UIImage) + ImageIO for HEIC/JPEG/PNG IO + IPTC keyword tagging |
| Persistence              | SwiftData for metadata, FileManager for blobs       |
| Logging                  | OSLog with subsystem `com.cyrus.musicwallpaper`     |
| Testing                  | XCTest + Swift Testing (new `@Test` macro)          |
| Dependency injection     | Plain protocol + initializer injection (no SPM DI)  |
| Secrets                  | `Secrets.xcconfig` (gitignored), surfaced via Info.plist |

**Minimum deployment target:** iOS 18.0. We rely on the latest App Intents, MusicKit, and Swift 6 strict concurrency.

**No third-party packages required for v1.** If you reach for one, justify it in the PR.

---

## 7. Permissions & Privacy

Up front, on first launch, request:
1. **Apple Music / MediaPlayer** — `MPMediaLibrary.authorizationStatus`, `MusicAuthorization.request()`.
2. **Photos add-only** — `PHPhotoLibrary.requestAuthorization(for: .addOnly)`. Only requested if the user enables the Photos Album output channel; skip otherwise.
3. **Background refresh** — enabled by default; the app explains it but doesn't gate on it.
4. **Notifications** — optional, only used to nudge user if Shortcuts automation breaks.
5. **SMTP credentials (Keychain)** — only if the user enables the Email Trigger channel. Stored locally, never leaves the device except as an outgoing TLS connection to the user's own mail server.

Never read user library contents beyond now-playing metadata. Never send track names to a server except as the literal query string to Last.fm / Google. All caches are local-only. The SMTP path connects only to the user-supplied host. Document all of this in the Privacy Manifest (`PrivacyInfo.xcprivacy`).

---

## 8. Testing as a User

You're going to be your own QA. Run this checklist after every meaningful change. **Do not skip steps** — the failure modes are subtle (race conditions on track change, cache staleness, image quality, photos permission edge cases).

### 8.1 Smoke test (every build)
1. Launch app on a real device (Simulator can't run MusicKit fully).
2. Confirm onboarding shows, grant all permissions.
3. Open Apple Music, play any popular song with clean album art.
4. Return to wallpaper app — within 2 seconds it should show "Now showing: <Track>" and the resolved image in the preview.
5. Check Photos → "Music Wallpaper" album — newest image should match.
6. Lock the phone, look at Lock Screen — wallpaper should be updated (if Shortcuts automation is configured).

### 8.2 Edge cases (run before each release)
- **Obscure track**: play something from a small artist not on Last.fm. Confirm fallback chain reaches Google or gradient and never crashes.
- **Rapid skip**: skip tracks 5 times in 10 seconds. Confirm no leaked tasks, final wallpaper matches the *settled* track, not an in-flight one.
- **Airplane mode**: kill network mid-song change. Confirm disk cache covers it or fallback gradient is shown.
- **Offline → online**: enable airplane mode, play 3 cached songs, disable airplane mode. Confirm prefetch resumes.
- **Pause/resume**: pause for 5 minutes, resume. Wallpaper should not change on pause; should refresh on resume only if track changed.
- **Long session**: play continuously for 1 hour. Memory should stay under 80 MB. Disk cache should not exceed 500 MB.
- **App killed**: swipe up to kill app. Play a new song. Confirm Shortcuts automation still triggers (App Intent must work without app foregrounded).
- **Custom override**: drop an image into the user override folder for an album you're about to play. Confirm it wins over Apple's artwork.
- **Reordered priority**: in Settings, move Last.fm artist to position 4 (above Apple Music). Play a track. Confirm the artist photo is what gets written.
- **Disabled source**: turn Google CSE off. Play a track with no Apple/Last.fm artwork. Confirm fallback gradient is shown without an outbound Google request (verify via Charles or Console).
- **30s timeout**: with a network conditioner set to "100% loss" on `ws.audioscrobbler.com`, play a track. Confirm the resolver gives up on Last.fm at ~30 s and moves on. No spinner, no hang in the UI.
- **Email channel**: enable Email Trigger with valid SMTP creds. Play a track. Confirm an email arrives within 30 s with subject `MusicWallpaper :: <artist>` and the correct image attached. Confirm the user's Shortcut runs and the wallpaper is set.
- **Email channel — bad creds**: enter intentionally wrong password. Confirm "Send test email" surfaces a clear error and the channel disables itself until the test passes again.
- **Email cooldown**: skip 10 tracks in 30 s with email enabled. Confirm only one email is sent (for the settled track) — not ten.
- **Artist tagging**: open Photos, search the artist name. Confirm only that artist's wallpapers appear. Confirm sub-album exists.

### 8.3 Visual quality bar
- No upscaling beyond 2x — if source is < device width / 2, prefer fallback.
- Letterboxing is forbidden; always crop to fill.
- Brightness check: if image average luminance > 0.85, render a subtle bottom gradient overlay so the clock stays readable.

---

## 9. Writing Clean Code (House Rules)

These rules are non-negotiable. The agent (and you) will follow them in every PR. They're chosen because they prevent the specific footguns this app has.

### 9.1 Concurrency
- **Swift 6 strict concurrency, no warnings.** If a warning appears, fix it — don't `@unchecked Sendable` your way out.
- All long-lived state lives behind an `actor`. Examples: `ImageCache`, `UserOverrideStore`, `ResolverPipeline`.
- UI types are `@MainActor`. Never call UIKit / SwiftUI from a background task.
- Every `Task { }` must be either bound to a view's `.task` modifier (so it cancels with the view), stored as a `Task` property and cancelled in `deinit`, or `Task.detached` with an explicit reason in a comment.

### 9.2 Architecture
- Pattern: **MV** (Model-View, using `@Observable`). No ViewModels-as-classes-of-one. Use `@Observable` macro from Observation framework.
- Each feature is a folder under `Features/` containing its View, its `@Observable` state container, and its tests.
- Cross-feature shared services live under `Services/` and are protocol-fronted so tests can inject fakes.
- No singletons except `Logger.shared` (OSLog wrapper). Everything else is injected.

### 9.3 Naming
- Types: `UpperCamelCase`, noun phrases. Protocols are *capabilities* (`ArtworkProviding`, not `ArtworkProtocol`).
- Methods: `lowerCamelCase`, verb phrases. Async methods that may throw are named for the result (`fetchArtwork`, not `tryToFetch`).
- Files: one primary type per file, named after that type.

### 9.4 Error handling
- Define `enum WallpaperError: Error` with cases for every failure mode shown to the user. Never `throw NSError` or string errors.
- Network errors are retried up to 2 times with exponential backoff before falling through to the next provider.
- Never crash on missing artwork. The fallback gradient is the contract.

### 9.5 Logging
- Use `Logger(subsystem: "com.cyrus.musicwallpaper", category: "<feature>")`.
- Log at `.debug` for trace, `.info` for state transitions, `.error` for handled failures.
- Never log full track titles at `.info` or above — they may be considered personal data. Hash track IDs in production builds.

### 9.6 Comments
- Code says *what*. Comments say *why*. No comments that restate the next line.
- Public types and async methods get doc comments (`///`). Private helpers usually don't need them.
- `// TODO:` and `// FIXME:` must include your initials and a date, e.g. `// TODO(cy 2026-05-17): handle empty queue`.

### 9.7 Tests
- Every Services/ type has unit tests with mocked dependencies.
- Every provider has a recorded-fixture test (a captured JSON / image so tests don't hit network).
- Integration tests for the resolver pipeline that prove the priority chain in section 4.
- UI tests are minimal — only the onboarding flow.

### 9.8 Git
- Trunk-based. Branches: `feat/<short>`, `fix/<short>`, `chore/<short>`.
- Commit messages: `<scope>: <imperative summary>` (e.g. `resolver: fall through to Last.fm on empty Apple artwork`). No multi-purpose commits.
- Every PR maps to one task ID from `TASKS.md`. The task ID goes in the PR title.

### 9.9 What "done" means for any task
A task is done when:
1. Code compiles with no warnings under Swift 6 strict concurrency.
2. Unit tests for the touched module pass.
3. The smoke test in 8.1 still passes on device.
4. The PR description includes a screenshot or short screen recording if UI changed.
5. The task's checkbox in `TASKS.md` is ticked, with a one-line note about anything surprising.

---

## 10. Files in This Folder

- `OVERVIEW.md` — this file. The "why" and the rules.
- `TASKS.md` — the bite-sized, ralph-loop-ready task list. Work it top to bottom.

When you (or an automated loop) pick up work, read this file, then open `TASKS.md`, find the next unchecked task, and execute exactly one task per loop iteration.
