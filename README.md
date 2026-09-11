# RepPower
(*RepSpeed was the old name seen in video)

An iOS app that measures how fast you're actually lifting. Put in your AirPods, start a set, and every rep gets a mean concentric velocity in m/s — spoken out loud as you rack it, then charted across the set so you can see exactly where you slowed down.

**Status:** Beta — currently runs through Xcode on my own device. App Store release planned.

## A set in progress (listen to the speed of each rep) ## 

https://github.com/user-attachments/assets/3b9328ba-c0f0-481e-a3e0-c75b8b00ae77

---

## Why I built it

Getting more powerful isn't just about load — it's load *and* speed. But in the gym you only ever really know one of those. The weight on the bar is a number; how fast you moved it is a feeling, and a feeling is a terrible way to tell whether you're improving. I wanted to stop going off vibes and actually see whether I was getting faster.

Velocity-based training gear exists, but it's a barbell sensor that costs a few hundred dollars. Then I found out the AirPods already have an accelerometer and gyroscope in them, streaming through Core Motion — hardware I was already wearing. The whole project became a calibration problem instead of a hardware one:

1. **Get vertical acceleration out of a sensor on your head** — project the acceleration onto the gravity vector so the reading means "up" regardless of how you're facing.
2. Integrating acceleration gives velocity, but integrating sensor bias gives **drift that grows forever** → learn the zero-point error from the moments you're provably standing still.
3. A number that arrives after you look at your phone is a number you ignore → **speak each rep** through the AirPods the moment you lock out.
4. Velocity alone doesn't tell you if the rep looked right → **film the set** with the front camera so the numbers have footage next to them.
5. Trusting a black box is its own problem → **export every raw sample as CSV**, so the math can be checked offline instead of believed.

---

## Features

### Per-rep velocity, spoken

Each rep produces mean velocity (the metric VBT actually trains off), peak velocity, time under tension, and distance travelled. Mean velocity is read aloud through the AirPods as soon as the rep is confirmed, so the feedback lands between reps instead of after the set.

### Velocity loss across a set

A finished set gets a bar chart of mean velocity per rep, with a dashed reference line at rep one so the drop-off is visible without reading numbers. Velocity loss — best rep versus worst, the standard fatigue measure — is called out separately, and turns red past 20%.

### Filming, without losing the voice

The front camera records one video per set, started and stopped with the set, so you can watch the lift alongside the numbers. It's deliberately **video-only with no audio track**: adding a microphone input hands the audio session to `AVCaptureSession` and the spoken feedback dies. Silent videos or spoken reps — the app picks spoken reps.

### Raw data export

With recording on, every sample is logged and written to CSV when the set ends — raw acceleration, the gravity vector, the derived vertical acceleration, and the integrated velocity and position. Shareable straight out of the set view, so any rep can be re-derived from scratch.

### Live readout

Three numbers update while you lift: current velocity, how far you've risen this rep, and the running **drift** estimate. That last one is a diagnostic — it should settle after a couple of reps, and if it wanders every rep, your AirPods are shifting in your ear.

---

## Tech stack

| | |
| --- | --- |
| Language | Swift |
| UI | SwiftUI |
| Motion | Core Motion — `CMHeadphoneMotionManager` |
| Camera | AVFoundation — `AVCaptureSession` |
| Voice | `AVSpeechSynthesizer` |
| Charts | Hand-drawn SwiftUI (`GeometryReader` + `Path`) |
| Export | CSV to the app's Documents directory |

---

## How it works

**The measurement chain.** AirPods stream `userAcceleration` with gravity already removed, plus a gravity vector. Projecting one onto the other gives acceleration along the vertical axis, signed so up is positive. A light low-pass filter takes out sensor noise above the ~1 Hz of a rep, then it's integrated twice — acceleration to velocity to position:

```swift
let aRaw = verticalSign * ((ua.x*g.x + ua.y*g.y + ua.z*g.z) / gMag) * 9.81
velocity += (aFiltered - accelBias) * dt
position += velocity * dt
```

**Calibration has to be earned.** Each set opens with a one-second stillness check that only counts *quiet* samples — shuffling under the bar resets the collection and extends the wait rather than poisoning the estimate. A bad bias estimate is what starts every drift cascade, so it's worth blocking the set for a second to get it right.

**Drift is the entire problem.** Integrating a sensor whose zero point is slightly off means the error compounds every sample. The fix is zero-velocity updates: whenever you're provably at rest, velocity is snapped to zero and the accumulated error is fed back into the bias estimate. Two rules govern when that's allowed to happen, both learned the hard way:

- **Never use velocity to decide whether velocity is trustworthy.** Drift pushes velocity up; if that reads as "moving", movement blocks the correction and the drift grows unchecked. The correction is gated only on whether a rep is in progress.
- **Quiet acceleration doesn't mean stationary.** Constant speed produces no acceleration at all — an accelerometer can't tell a steady descent from standing still, any more than you can feel a plane's cruising speed. Zeroing on quiet acceleration alone injects your actual speed as a permanent offset.

**Rep boundaries come from the integral, not the acceleration.** A rep starts when velocity crosses a threshold — not when position bottoms out, because if you pause at the bottom your lowest point lands somewhere random in that pause. The top is called when the *rise over a trailing window* stops growing, which is immune both to acceleration going silent at constant speed and to a sticking point tripping a velocity threshold mid-climb. Candidates that come out too short, too small, too slow, or physically implausible are rejected and the velocity is zeroed on the way out.

**Keeping 25 Hz off the main thread.** Motion is delivered to a serial background queue that owns all the integration state; nothing crosses to the main thread except through an explicit dispatch, at 12 Hz rather than per sample. The three live numbers live in their own `LiveReadout` observable, held by the tracker as a plain `let` and not a `@Published` property — if it were published, mutating it would notify every observer of the tracker and drag the whole screen into a redraw on every sample. That was the bug that froze the readout after the first rep.

---

## Status

RepPower is in beta. It lives in Xcode — I build it to my own iPhone and use it in the gym, which is where the tuning constants in `RepTracker.swift` came from. There's no public build yet: no App Store listing, no TestFlight, and nothing to install from here.

The plan is an App Store release once the rough edges below are sorted — chiefly saving sets between launches and removing the manual axis-sign step. Until then this README describes what's built and working rather than how to install it.

### What it runs on

| | |
| --- | --- |
| Hardware | iPhone + AirPods Pro, AirPods 3+, AirPods Max, or Beats Fit Pro |
| Xcode | 15 or newer |
| Permissions | Motion & Fitness, Camera |

Headphone motion isn't available in the simulator, so it only runs on a physical device.

---

## Known limitations / next up

- **Sets don't persist** — completed sets live in memory, so quitting the app loses your session. The per-set CSVs survive in the Documents directory, but the reps themselves need real storage. This is the main thing standing between the current build and a release.
- **Only motion-capable AirPods work** — it's `CMHeadphoneMotionManager` or nothing, so regular AirPods and third-party headphones aren't supported.
- **Vertical axis is assumed** — displacement is measured along gravity, so the numbers describe squats and presses well and anything with a horizontal component less well.
- **`verticalSign` is a hardcoded constant** — if live velocity swings negative when you rise onto your toes, it has to be flipped by hand. Detecting the sign during calibration would remove the manual step.
- **Videos are silent by design** — a microphone input would take the audio session away from the spoken rep feedback. Getting both would mean routing capture audio through the same session rather than letting AVFoundation claim it.
