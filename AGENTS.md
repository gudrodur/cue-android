# Cue for Android

Android port of Jokull Sólberg's Cue, an iPhone teleprompter that follows the speaker's voice
and burns word-by-word captions into the recorded video. This is a fork of `jokull/cue`,
made with his blessing: the SwiftUI app, the Swift core, the Cloudflare Worker and the site
are the reference implementation, and the Android client (Kotlin, Jetpack Compose, CameraX,
Media3) is a port in progress. Public.

## Stack and entry points

- `App/` — the iOS app (SwiftUI, iOS 26); `project.yml` generates the Xcode project.
- `Packages/CueCore/` — the testable Swift core: tracking, pace, alignment, captions, export.
- `api/` — `cue-api`, the Worker that holds the ElevenLabs key, verifies the StoreKit
  subscription and proxies batch transcription. Shared as-is with iOS.
- `site/` — cue.solberg.is static site. `branding/` and `research/` are shared material.
- `android/` — not started yet; the target architecture is in the README.

## Build, test and run

```sh
# Worker and site (Node + npm)
cd api && npm ci && npm test
cd api && npm run dev            # wrangler dev
cd site && npm ci && npm run dev

# Swift core (macOS with Xcode)
cd Packages/CueCore && swift test
```

The Android app has no Gradle project yet. Not run here: the Worker tests need an npm install
and the Swift suite a macOS toolchain. The repo has no CI, so run the suite for the layer you
touched before opening a PR; `docs/TESTING.md` maps each layer to its test.

## Deploy and accounts

- `api/` and `site/` deploy with `wrangler deploy` on the upstream Cloudflare account. The
  operator needs the Cloudflare credentials and the Worker secrets (`ELEVENLABS_API_KEY`, app
  tokens, subscription config).
- No GCP project. `Config/Cloud.xcconfig` and `.env.example` carry non-secret settings;
  never commit real keys or an Apple/Google purchase credential.

## Rules

- Keep the name, icon and visual identity, the word-alignment model and the two Worker routes
  shared with upstream. The Android client speaks the same endpoints with the same token and
  metering semantics.
- The Swift tree is the behavioural reference, not dead code: port the tracker constants and
  alignment rules from it rather than restating them.
- Audio is never stored and a realtime token is single-use. Do not add a route that keeps
  either.
- Develop the fork on its own `main`; do not rewrite upstream history.

## Docs

`README.md` (status, layout, target architecture), `docs/TESTING.md`,
`research/voice-tracking.md`, and the `api/` and `site/` sources.
