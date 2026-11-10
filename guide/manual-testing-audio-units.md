# Manual Test Plan — Audio Unit Effects

CI covers the Audio Unit code offline (scanner, host, chain, persistence, host reuse) with synthetic buffers. It cannot exercise real process taps, real output devices, Bluetooth, sleep/wake or third-party plug-ins. This checklist covers that gap. Run it on a real Mac before merging.

## Setup

1. **Requirements:** macOS 15 or later, Xcode, at least two output devices (e.g. built-in speakers + wired or Bluetooth headphones). A third device or an HDMI display is a bonus.
2. **Quit any installed FineTune.** The dev build shares its settings with the release app.
3. **Back up your settings:** copy `~/Library/Application Support/FineTune/settings.json` somewhere safe. Restore it when you are done.
4. Check out the `feature/audio-unit-chain` branch, open `FineTune.xcodeproj`, then build and run (⌘R).
5. **Follow the logs:** open Console.app and filter on subsystem `com.finetuneapp.FineTune`, categories `AUEffectChain` and `AUEffectHost`. Each chain rebuild logs `Created AU effect chain with X/Y plugins at R Hz (N reused)`.
6. Use a music app (Music, Spotify) and a browser tab playing audio as test sources.

Apple's built-in effects are enough for every test: `AUNBandEQ`, `AULowpass`, `AUDelay`, `AUReverb2`, `AUDynamicsProcessor`, `AUPeakLimiter`.

## Where the controls are

- **Per-app effects:** expand an app row (EQ button). The **Effects** section sits under the EQ panel.
- **Per-device effects:** click **FX** on an output device row.
- **In a chain:** **Add Effect**, an on/off switch per effect, a chain-level switch (shows **Bypassed**), up/down arrows to reorder, ✕ to remove, a factory preset picker when the plug-in has presets, and **Open Custom Interface** / **Open Generic Interface**.

## 1. Discovery

- [ ] **Add Effect** lists the Apple effects, grouped by manufacturer and sorted.
- [ ] Search filters the list. Favorites can be toggled and persist after a relaunch.

## 2. Per-app chain basics

Play music in one app and keep a second app playing for comparison.

- [ ] Add `AULowpass` and open its interface. Lowering the cutoff audibly muffles **only** that app.
- [ ] The per-effect switch off/on toggles the effect without a click or dropout.
- [ ] The chain switch shows **Bypassed** and the audio returns to dry.
- [ ] Add `AUDelay`, then pick a factory preset. The sound changes accordingly.
- [ ] Reorder (`AULowpass` ↔ `AUDelay`). The order is audible: delayed echoes filtered or not.
- [ ] Remove each effect. The audio returns to dry with no glitch.
- [ ] **Open Custom Interface** and **Open Generic Interface** both open a usable window. Parameter moves are heard live.

## 3. Host reuse (this branch)

Editing a chain must not reset the effects that did not change.

- [ ] Add `AUReverb2` with a long decay. Play a short sound, and **while the tail rings**, add a second effect. The tail keeps ringing instead of cutting.
- [ ] With `AUReverb2`'s window open, toggle **another** effect on/off. The window stays open and keeps controlling the sound.
- [ ] Console shows `(1 reused)` (or more) for these edits.
- [ ] Changing `AUReverb2`'s factory preset rebuilds it: Console shows `0 reused` for that entry, and the preset applies.

## 4. Persistence

- [ ] Quit and relaunch FineTune. App chains, device chains, order, on/off states, bypass and presets are restored.
- [ ] Quit and reopen the music app. Its chain comes back as soon as it plays.
- [ ] Parameter tweaks made in a plug-in window survive a relaunch.

## 5. Per-device chains

- [ ] Add `AUPeakLimiter` or `AUDynamicsProcessor` with **FX** on the current output. It applies to every app on that device.
- [ ] App and device chains stack: the app chain runs first, then the device chain.
- [ ] A chain set on device A does not apply when audio plays on device B.

## 6. Switching output devices

Keep effects active on the app **and** on both devices.

- [ ] Switch speakers → wired headphones while playing. The transition is smooth and the destination device's chain applies.
- [ ] Switch to Bluetooth headphones (A2DP) and back. No silence, no stuck bypass, effects still active.
- [ ] Edit a chain *during* a switch: rapidly toggle an effect while changing device. No crash, no stuck state.
- [ ] Unplug the active headphones while playing. The fallback device plays with its own chain.

## 7. Formats and topology

- [ ] In Audio MIDI Setup, change the output's sample rate (e.g. 44.1 → 96 kHz) while playing. Effects keep working (they are rebuilt; Console shows the new rate).
- [ ] Multi-device output, or an HDMI/multichannel device: audio plays **unprocessed** but intact (known limitation — effects only run on stereo). No crash, no silence.

## 8. Robustness

- [ ] Sleep the Mac for a few minutes while playing, then wake. Audio and effects resume.
- [ ] Load 4–5 effects on one app. No dropouts. Note FineTune's CPU in Activity Monitor.
- [ ] **Third-party plug-in (optional):** add one, use its custom interface, relaunch, and check that its state is restored.
- [ ] **Missing plug-in:** remove a third-party plug-in from `~/Library/Audio/Plug-Ins/Components` and relaunch. The entry shows the warning triangle ("Failed to load — plugin may be missing or incompatible") and the rest of the chain still works.
- [ ] **Crash protection:** if a plug-in crashes FineTune, it is flagged after relaunch ("This plugin may have caused a crash") and disabled.

## 9. Regression (no effects)

- [ ] With no effects anywhere, volume, mute, boost, EQ, AutoEQ, loudness and device routing behave exactly as on `main`.
- [ ] Apps without a chain are unaffected while another app has one.

## Reporting

For each failure, note: macOS version and Mac model, output devices involved, plug-ins and chain order, steps to reproduce, and the related Console lines (`AUEffectChain` / `AUEffectHost`).
