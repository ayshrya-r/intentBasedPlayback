# IntentPlayer — tvOS POC

A fully working tvOS sample app demonstrating **Intent-Based Playback Controls** on Apple TV.

## What It Does

Instead of mapping Siri Remote swipes to fixed seek increments, this app infers *what the user is trying to accomplish* and responds accordingly:

| Swipe | Inferred Intent |
|-------|----------------|
| Slow swipe | `SEEK_SMALL` — precise ~2–8s skip |
| Medium swipe | `SEEK_MEDIUM` — ~10–30s skip |
| Fast swipe | `SEEK_LARGE` — up to 8% of content duration |
| Fast swipe near chapter | `CHAPTER_JUMP` — snaps to boundary |
| Paused + swipe (frame mode) | `FRAME_STEP` — single frame advance |
| Hold + swipe | `SPEED_ADJUST` — 0.5× or 2× rate |

---

## Requirements

- Xcode 15+
- tvOS 17.0+ deployment target
- Apple TV (physical device) or tvOS Simulator
- Active internet connection (streams sample MP4s from Google CDN)

---

## Setup

1. **Open project**
   ```
   open IntentPlayer.xcodeproj
   ```

2. **Set team** in Target → Signing & Capabilities → Team (use your Apple ID)

3. **Select scheme** → IntentPlayer → Apple TV Simulator or your device

4. **Run** (⌘R)

---

## Architecture

```
Siri Remote Touchpad
       │
       ▼
SwipeGestureHandler          ← collects 80-150ms input window
       │  InputBundle
       ▼
IntentClassifier             ← rule-based intent tree
       │  ClassifiedIntent + confidence
       ▼
Confidence Gate              ← drops below-threshold intents → fallback
       │  gated PlaybackIntent
       ▼
AdaptiveDeltaCalculator      ← velocity^1.6 × contentLength × boundaryFactor
       │  seek delta in seconds
       ▼
PlaybackIntentDispatcher     ← calls AVPlayer.seek() with correct tolerance
       │
       ▼
    AVPlayer
```

### Key Files

| File | Purpose |
|------|---------|
| `PlayerViewController.swift` | Root VC, wires all components |
| `PlaybackModels.swift` | All enums and data types |
| `IntentClassifier.swift` | Rule tree: velocity → intent + confidence |
| `AdaptiveDeltaCalculator.swift` | Non-linear velocity→delta math |
| `SwipeGestureHandler.swift` | Raw touchpad events → InputBundle |
| `PlaybackIntentDispatcher.swift` | Full pipeline orchestrator |
| `ChapterManager.swift` | Chapter boundary detection & navigation |
| `IntentOverlayView.swift` | On-screen intent confirmation badge |
| `DebugOverlayView.swift` | Real-time pipeline debug panel |
| `HapticFeedbackEngine.swift` | Taptic engine wrapper |
| `SampleMediaProvider.swift` | Sample HLS streams + synthetic chapters |

---

## Remote Controls

| Button / Gesture | Action |
|-----------------|--------|
| **Swipe left/right** | Intent-based seek |
| **Select (click)** | Play / Pause |
| **Play/Pause button** | Play / Pause |
| **Menu button** | Toggle debug overlay |

---

## Modes

Cycle through modes in `PlayerViewController.cycleMode()` or modify `playerConfig.mode`:

- **Standard** — full intent inference
- **Explore** — scrubbing with seek to position
- **Chapter Snap** — every swipe jumps to nearest chapter
- **Frame Step** — single-frame advance (requires player paused)

---

## Accessibility

Toggle `playerConfig.accessibilityConfig.isEnabled = true` to bypass all intent inference and use a fixed 10-second seek. This also enables `UIAccessibility` announcements.

---

## Adaptive Delta Formula

```swift
let norm     = min(velocity / 800, 1.0)
let scaled   = pow(norm, 1.6)                          // exponential curve
let maxSeek  = min(duration * 0.08, 120.0)             // content-aware cap
let boundary = dist < 15 ? (dist / 15.0) : 1.0        // dampen near chapters
let delta    = scaled * maxSeek * boundary             // signed by direction
```

---

## Seek Tolerance Strategy

| Intent | Tolerance |
|--------|-----------|
| `seekSmall/Medium/Large` | ±0.5s (fast response) |
| `chapterJump` | `.zero` (exact frame) |
| `frameStep` | `.zero` (exact frame) |
| `exploreMode` | ±1.0s (fine for scrubbing) |

---

## Extending

**Add a CoreML classifier** — replace the rule tree in `IntentClassifier.classify()` with an `MLModel` prediction using the same `InputBundle` features.

**Add thumbnail scrubbing** — in `PlaybackIntentDispatcher.perform()` for `.exploreMode`, use `AVAssetImageGenerator` to show preview frames.

**Add custom chapters** — call `chapterManager.setManualChapters()` with your own `[Chapter]` array, or let `chapterManager.loadChapters(from:)` read from `AVAsset` timed metadata.
