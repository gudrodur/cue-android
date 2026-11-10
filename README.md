# Cue for Android

An Android teleprompter that follows your voice in real time, then burns word-by-word captions
into the video you just recorded.

Hold the record button to film a clip; the script, under the camera, moves line by line as you read
it, and waits where you stopped. Clips stitch into one take; captions are timed to your actual words.
Works in Icelandic.

**Cue** is [Jökull Sólberg](https://github.com/jokull)'s app — an iPhone teleprompter, published at
[cue.solberg.is](https://cue.solberg.is). This repository is a fork of
[`jokull/cue`](https://github.com/jokull/cue), made with his blessing, to build the Android version.
It keeps the same name, icon and visual identity, the same word-alignment model, and the same
Cloudflare Worker in front of ElevenLabs. Only the client changes: SwiftUI becomes Kotlin and Jetpack
Compose.

## Status

This is a **port in progress**. Read the tree as two things at once:

| | State |
|---|---|
| **Reference implementation** | Complete and shipping: `App/` (SwiftUI, iOS 26), `Packages/CueCore/` (Swift), the `cue-api` Worker, the marketing site. Read it, run its tests, use it as the specification. |
| **Android app** | Not started. No Gradle project, no Kotlin sources yet. The target architecture is sketched below; the first commit will scaffold `android/`. |

The Swift code stays in the tree as the behavioural reference: the tracker's tuning constants, the
alignment rules, the caption layout and the clip model are the parts worth porting faithfully, and
they are cheaper to read as working code than to restate as prose.

## How it works

Two speech passes, one script.

1. **Live tracking.** While you record, a realtime speech stream produces words as you say them.
   `ScriptTracker` aligns the last handful of heard words against a window of the script around the
   cursor — local alignment, online, forgiving of misheard words. The match must end at what was
   *just* heard; a lone word can nudge the cursor, a long jump needs several matches, and the cursor
   never moves backwards, so an ad-lib holds your place instead of skipping ahead.
2. **Word alignment.** After the clips are stitched into a take, `Aligner` does global sequence
   alignment (Needleman–Wunsch) of the script against the take's transcript. Paired words get the
   script's spelling and the voice's timing; unpaired heard words are ad-libs; unpaired script words
   were skipped or missed, and short runs are interpolated rather than dropped.
3. **Captions and export.** `CaptionLayerBuilder` and `CaptionExporter` render the captions as a
   layer over the video — cropped to 9:16, 4:5, 1:1 or 16:9, styled, optionally animated word by
   word — and write a new file. The take itself is never modified.

The interesting part is (1): a prompter that lags feels broken and one that runs ahead is worse, so
the cursor is deliberately conservative. `research/voice-tracking.md` explains the measurements and
why the thresholds are what they are.

## The Cloudflare Worker

`api/` is `cue-api`, a Worker that holds the ElevenLabs key so no client ever ships it:

| Route | What |
|---|---|
| `POST /v1/realtime-token` | Mints a **single-use** Scribe v2 Realtime token (one WebSocket session, 15 min). The app then connects to ElevenLabs directly — audio never passes through the Worker. |
| `POST /v1/transcribe?language=is&keyterm=…` | Proxies a finished take's audio (max 8 MB / 3 min) to Scribe v2 batch, returns `{ language, words: [{ text, start, end, logprob }] }`. |

Both need `Authorization: Bearer <app token>` and, when `REQUIRE_SUBSCRIPTION` is on, a current
subscription: `X-Cue-Transaction` carries an Apple StoreKit signed transaction, which the Worker
verifies against Apple's certificate chain and meters per subscriber per month. Audio is never
stored. Rate limits are per IP and per subscription.

Keeping the Worker shared is the point of reusing the upstream repository: the Android client speaks
the same two endpoints, with the same token and metering semantics. The one piece that is iOS-only is
purchase verification — `api/src/apple.ts` checks App Store transactions, so Android needs a Google
Play Billing equivalent in the same Worker before subscriptions can be sold there.

## Layout

| Path | What | Status |
|---|---|---|
| `App/` | The iOS app (SwiftUI, iOS 26). `project.yml` generates the Xcode project. | Reference |
| `Packages/CueCore/` | Testable Swift core: script tracking, reading pace, alignment, captions, export, clips. `swift test` runs offline on macOS. | Reference — the port's specification |
| `api/` | `cue-api` Worker: holds the ElevenLabs key, checks the subscription, mints realtime tokens, proxies batch STT. | Shared as-is |
| `site/` | cue.solberg.is (static, Cloudflare). | Upstream |
| `branding/` | Icon (`App/Cue.icon`), App Store frames (`frames.py`), Blender scripts. | Shared identity |
| `research/` | Speech research and the eval tools (`scribe_eval.py`, realtime recorder, `tracklag` replay). | Shared |
| `docs/TESTING.md` | How to test each layer, from unit tests to the phone. | Reference |
| `android/` | The Android app: Kotlin, Jetpack Compose, CameraX, Media3. | **Not yet written** |

## Target architecture for Android

The mapping is mechanical where it can be, and a real decision where it cannot:

| iOS | Android |
|---|---|
| SwiftUI views (`RecordView`, `PrompterView`, `ExportSheet`, `TakeView`) | Jetpack Compose, same screen structure and prompter layout |
| `AVCaptureSession` / `AVAssetWriter` (`CameraRecorder.swift`) | CameraX `VideoCapture` with `Recorder`, one file per clip |
| `AVMutableComposition` (`ClipStitcher`) | Media3 `Transformer` composition of the clip sequence |
| `AVVideoComposition` + `CALayer` captions (`CaptionLayerBuilder`) | Media3 `Transformer` + `Effect`/`OverlayEffect`, or a Canvas-drawn `TextureView` composited during export |
| `SFSpeechRecognizer` / `SpeechAnalyzer` (`Speech.swift`) | ElevenLabs Scribe v2 Realtime over WebSocket, plus an on-device recognizer for locales it covers |
| `CueCore` (Swift, pure logic) | Kotlin port of the same types, with the same tests — the alignment and tracker suites are the acceptance criteria |
| StoreKit subscription | Google Play Billing, verified in `cue-api` |

The port keeps the layering: prompter/tracking logic is pure Kotlin with no Android dependencies, so
it is unit-testable on the JVM the way `CueCore` is testable on macOS. Anything that touches a
camera, a decoder or a canvas stays behind a thin interface — that is what makes `tracklag`-style
replay of real takes possible when retuning the tracker.

## Running the reference implementation

macOS only; the Android build does not exist yet.

```sh
brew install xcodegen
cp .env.example .env            # ELEVENLABS_API_KEY, CUE_API_URL, CUE_API_TOKEN, CUE_DEV_TOKEN
scripts/cloud-config.sh         # writes the gitignored Config/Cloud.secrets.xcconfig
xcodegen generate && open Cue.xcodeproj
(cd Packages/CueCore && swift test)
```

The Worker runs on its own:

```sh
cd api && npm install && npm test
npx wrangler dev --port 8788    # secrets in api/.dev.vars, gitignored
scripts/test-api.sh             # health, auth gates, real transcription
```

Speech runs on ElevenLabs Scribe (realtime for the prompter, batch for captions) through the Worker;
see `research/elevenlabs-api.md` and `research/voice-tracking.md` for why. Cue is a free manual
prompter (the thumb scrolls the script). **Cue Voice** — the voice-following script and the captions
— is a yearly subscription on iOS, which exists because the speech service costs money per minute;
the Android tier will use Play Billing against the same Worker.

## Staying in sync with upstream

```sh
git remote add upstream https://github.com/jokull/cue.git
git fetch upstream && git merge upstream/main
```

Changes that belong to the iOS app or the shared model go upstream as pull requests rather than
living only here — the tracker and the Worker are worth keeping in one place.

## Credits

The app, its design, the alignment approach and the ElevenLabs research are
[Jökull Sólberg](https://github.com/jokull)'s work: [`jokull/cue`](https://github.com/jokull/cue),
[cue.solberg.is](https://cue.solberg.is). The Android port is by
[Guðröður Atli Jónsson](https://github.com/gudrodur), with his permission, under the same name.
