# DolphiniOS 5.0.0b6 — iOS 26 / iOS 27 Controller Fix (unofficial build)

An unofficial, patched build of **DolphiniOS 5.0.0b6** that fixes physical game
controllers being **dead in-game on iOS 26 / iOS 27**, even though they map and
respond correctly in the app's settings.

> **Status:** ✅ Fixed and verified on a real device (DualShock 4, iOS 27 beta, with JIT).
> Buttons, sticks, triggers, D-pad, **and gyro** all reach the emulator during gameplay.

---

## TL;DR

| | |
|---|---|
| **Symptom** | A paired controller (e.g. DualShock 4) maps fine and shows live input in DolphiniOS's controller settings, but does **nothing once a game starts**. Touch controls keep working. |
| **Cause** | On iOS 26/27, **Game Mode** captures the controller's entire input feed for the system the moment a game goes full-screen, so the emulator core never receives it. |
| **Fix** | Two `Info.plist` flags are disabled so the app receives raw controller input directly during gameplay. |
| **Result** | Confirmed in-game: the core polls the pad ~875×/sec and every input (incl. gyro) registers. |

Grab the latest `DolphiniOS-fix-ios.ipa` from the [**Releases**](../../releases) page.

---

## Install

1. Download the newest **`DolphiniOS-fix-ios.ipa`** from [Releases](../../releases).
2. Sideload it with **SideStore** or **AltStore**. The IPA is **unsigned** — your sideloader re-signs it on install with your own certificate.
3. For full-speed emulation you need **JIT**. Enable it the usual way for your setup (e.g. **StikDebug** / SideStore). *(Confirmed working on iOS 27.)* Without JIT you can still run games in **Cached Interpreter** mode, but it's slow.
4. Set up your controller as normal: **Controllers → Port 1 → Standard Controller**, pick your controller under **Device**, and map the buttons. It now works in-game.

---

## The bug, in detail

DolphiniOS reads a physical controller through Apple's **GameController** framework.
On iOS 26/27 the following happened with a Bluetooth controller such as a DualShock 4:

- In **Controllers → mapping / input display**, every button, axis, the touchpad and the gyro showed live values and mapped correctly.
- The moment a **game launched**, the controller went completely silent. Touch input still worked, and the only buttons that did anything were **PS** (opened the iOS game overlay) and **Share** (started a screen recording) — both of which are reserved by iOS itself.

This "works in menus, dead in-game" split is the key clue, and it's why the bug is so confusing: the controller is clearly paired and functional, yet the game receives nothing.

---

## Root cause

DolphiniOS's app declares two keys in its `Info.plist`:

```xml
<key>GCSupportsControllerUserInteraction</key> <true/>
<key>GCSupportsGameMode</key>                  <true/>
```

`GCSupportsGameMode = true` opts the app into **iOS Game Mode**. When a game goes
full-screen on iOS 26/27, Game Mode (together with the "press the PS button for the
game overlay" system layer) **takes over the controller's input feed for the system**.
The app's own reads of the controller then return nothing — so the emulator core,
which polls the controller every frame, sees a flatlined device.

The settings screens still worked because **Game Mode is not active there** — it only
engages once a game is running full-screen. That's the whole asymmetry.

### Why we know it's Game Mode and not the UIKit focus engine

The other flag, `GCSupportsControllerUserInteraction`, can also divert controller input —
but only **navigation** inputs (buttons, D-pad, sticks) that the UIKit focus engine uses
to move selection around. It has no use for **motion data**. In the broken state the
**gyro died in-game too**, which means the *entire* feed was being captured at the
system level — the signature of Game Mode, not the focus engine.

---

## The fix

The patch flips both keys in `Source/iOS/App/DolphiniOS/Info.plist` to `false`:

```xml
<key>GCSupportsControllerUserInteraction</key> <false/>
<key>GCSupportsGameMode</key>                  <false/>
```

- **`GCSupportsGameMode = false`** — the operative fix. Stops iOS from capturing the controller for Game Mode while a game is running.
- **`GCSupportsControllerUserInteraction = false`** — stops the UIKit focus engine from routing controller input to the responder chain instead of the app.

The evidence points at Game Mode as the actual culprit, but **both** are disabled in this
build for certainty.

### Trade-offs

- You can no longer use the controller to navigate **DolphiniOS's own menus** (use touch for menus). In-game play is unaffected.
- Game Mode's CPU/GPU scheduling and Bluetooth-latency boost are off.

Neither matters for actually playing games. If you'd prefer to keep controller-driven
menu navigation, a **Game-Mode-only** variant (leaving `GCSupportsControllerUserInteraction`
enabled) is possible — open an issue and it can be built.

---

## How it was diagnosed

Rather than guess, a one-off **diagnostic build** instrumented `MFiController`
(the iOS controller backend in Dolphin) to log, straight from the GameController framework
on the core's input thread:

- `getstate_calls` — how many times the emulator core actually read the controller's bindings,
- the live raw values of every button / stick / trigger / D-pad / **gyro**, every ~0.4 s,
- controller connect / disconnect (`[device CREATED]` / `[device DESTROYED]`) events.

The on-device log (`input_diag.log`) showed, **in-game**:

```
getstate_calls climbing ~875/sec   →  the core IS polling the pad every frame
A=1.00 … B=1.00 … X=1.00 … Y=1.00  →  face buttons register
LT=1.00 / RT=1.00, dpad=0/1/0/0    →  triggers and D-pad register
motion(active=1 rot=0.94/-5.04/…)  →  gyro streaming
```

That confirmed the controller's full feed reaches the emulator with the fix applied.
(Two things that *looked* suspicious turned out to be harmless: the controller reports
`playerIndex=-1` / `attached=0`, and it is briefly torn down and recreated at game boot —
the binding re-resolves cleanly either way.)

The production build has this logging **removed**.

---

## How the build works (CI)

The workflow at [`.github/workflows/build-ios-ipa.yml`](.github/workflows/build-ios-ipa.yml)
builds the IPA from source on GitHub Actions:

1. Runs on **`macos-26`** (Xcode 26 / iOS 26 SDK) — required, because b6 uses iOS 26 APIs.
2. Clones [`OatmealDome/dolphin-ios`](https://github.com/OatmealDome/dolphin-ios) at tag **`v5.0.0b6`** with submodules.
3. Applies the **`Info.plist` controller fix**.
4. Applies two build-time workarounds needed for the Xcode 26 toolchain:
   - **fmt** — the bundled `fmt` library's `consteval` path doesn't compile under Xcode 26's Clang; patched to drop `consteval` (`FMT_CONSTEVAL`).
   - **bartycrouch** — a localization "Run Script" build phase calls `bartycrouch`, which isn't installed on CI runners; it's stubbed with a no-op (it only syncs storyboard strings — purely cosmetic).
5. Builds the **`DiOS (NJB)`** (non-jailbroken) scheme **unsigned**, packages an unsigned IPA, and publishes it to **Releases**.

### Build it yourself

1. Fork this repo.
2. **Settings → Actions → General → Workflow permissions → Read and write permissions.**
3. **Actions → Build DolphiniOS … → Run workflow.**

---

## Compatibility

- **DolphiniOS:** 5.0.0b6
- **iOS:** 26 / 27 (tested on iOS 27 beta)
- **Controllers:** DualShock 4 confirmed. Other MFi / Bluetooth controllers should benefit from the same fix, since the cause is system-wide, not controller-specific.

---

## Credits & license

- **DolphiniOS** by [OatmealDome](https://dolphinios.oatmealdome.me) — licensed **GPL-2.0-or-later**.
- **Dolphin Emulator** — <https://dolphin-emu.org>
- This is an **unofficial community build**. It is **not** affiliated with, supported by, or endorsed by OatmealDome or the Dolphin project. Please do **not** ask the upstream projects to support this build.
- The source and this patch are distributed under the same license as upstream (**GPL-2.0-or-later**).

## Disclaimer

This is an unofficial build targeting a beta operating system. Use at your own risk.
