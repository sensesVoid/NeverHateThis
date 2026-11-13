# PRD: NeverHateThis — Android Pedalboard & Amp App

Status: Draft v0.2 · Date: 2026-11-13 · Platform: Android · Core language: Rust · Brand: NeverHateThis

## 1. Summary

NeverHateThis is a lightweight, offline-first Android app for guitar, bass and vocal effects. Users build a signal chain (Rig) from pedals, an amp and a cabinet, browse a Pedal Collection, and create their own pedals in a Pedal Maker with custom cover art. The target audience is people exploring guitar effects who cannot afford an audio interface, so the app must work with cheap inputs (analog iRig-style cables, headset-jack adapters, the built-in mic) and run well on low- and mid-range phones. The goal is a quality of tone and usability that can rival existing apps such as ToneBridge.

## 2. Goals and non-goals

### Goals
- Playable, low-latency live effects on common Android phones, with honest latency reporting.
- Works with analog iRig-style inputs and USB audio interfaces, plus the built-in mic for vocals.
- Small install size and low CPU use; fully functional offline with no account.
- A Pedal Maker that lets users combine effect blocks into shareable pedals.
- Open-source DSP stack with clearly tracked licenses.

### Non-goals (v1)
- Live playing over Bluetooth input (not feasible, see section 7).
- Guaranteeing that a guitar or bass brand changes the real tone (see section 5.6).
- Cloud accounts, a social network, or an online marketplace.
- Plugin hosting (VST/CLAP/LV2) on Android.

## 3. Target users

| Persona | Needs |
|---|---|
| Budget beginner | Wants to try effects with a phone and a cheap cable; needs guidance, presets and a forgiving setup |
| Hobbyist guitarist/bassist | Wants decent amp tones, a tuner and a looper for practice |
| Singer / home recorder | Wants a vocal chain (gate, compressor, EQ, reverb, delay) on the built-in or a wired mic |
| Tinkerer | Wants to build and share custom pedals |

## 4. Success metrics

- Round-trip latency under 20 ms on supported USB paths; measured and shown on every device (target on analog/headset paths: as low as the device allows, always displayed).
- Zero audible dropouts (xruns) over a 10-minute session on the reference test devices.
- CPU use of the default chain under an agreed budget on the lowest reference device (to be set after the first prototype).
- Install size target: APK under ~25 MB without downloaded content (to be validated).
- First-run calibration completed successfully by most test users.

## 5. Features

### 5.1 Rig
- Ordered signal chain: input stage, pedals, amp, cabinet, output.
- Add, remove, reorder and bypass blocks; adjust knobs; save and load rigs.
- Hot-swapping blocks without clicks or dropouts (crossfade).
- Input and output level meters, noise gate on by default.

### 5.2 Amp
- Selectable amp models, in two tiers:
  - Standard (default): classic DSP preamp, tone stack, power-amp stage and cabinet impulse response; very low CPU.
  - HD (optional): small neural amp models for higher realism on capable devices.
- Cabinet selector with downloadable impulse responses.

### 5.3 Pedal Collection
- Built-in library of effects: noise gate, compressor, overdrive, distortion, fuzz, EQ, chorus, flanger, phaser, tremolo, delay, reverb, pitch/harmony, wah, plus vocal-oriented effects (de-esser, pitch correction).
- Each pedal has a short explanation of what it does and where it belongs in the chain.

### 5.4 Pedal Maker
- Combine effect blocks into a new pedal and choose which parameters appear as knobs.
- Save pedals as versioned, schema-validated data files (data only, never executable code) so import and sharing are safe.
- Cover art editor: pick an image, use simple templates, or generate art with an AI image provider using the user's own key (BYOK).
- Optional: describe a pedal in text and get a proposed preset, validated against the schema before loading.

### 5.5 Instruments and input
- Instrument modes: Vocal (mic), Electric Guitar, Bass.
- Input paths supported: analog iRig-style cable / headset-jack adapter, USB audio interface, built-in mic.
- First-run loopback calibration to measure real round-trip latency and select the buffer size automatically; stored per device and input route.
- Input stage with gain, DC blocking, high-pass filter and boost for weak headset inputs.
- Feedback guard warning when the mic is live with speakers.

### 5.6 Instrument voicing profiles
The instrument's real tone is determined before the signal reaches the app, so the app cannot reproduce a specific brand's sound. Instead, offer pickup-style profiles (for example single-coil, humbucker, P-bass, J-bass) that adjust input EQ and gain staging. Example brands may be used as labels, and the feature is described honestly in the UI as an input voicing, not a brand emulation.

### 5.7 Learning and utility tools
- Tuner, metronome, looper, and a backing-track player.
- Tone recipes for popular styles, loadable in one tap.
- Signal-chain explainer for beginners.
- Record-then-reamp mode: record the dry signal, then process it offline or with any playback route.

## 6. Technical approach

### 6.1 Architecture
- Rust core engine (DSP, chain, parameters) exposed to a Kotlin + Jetpack Compose UI through UniFFI or the `jni` crate.
- Audio I/O through Oboe (via `oboe-rs`) using AAudio low-latency mode, 48 kHz, small buffers, unprocessed input preset.
- Real-time rules: no allocation, locks or I/O on the audio thread; parameter changes via lock-free queues or atomics; parameter smoothing everywhere.
- Common effect trait: process, set parameter, reset. Chain changes are built off-thread and crossfaded in.

### 6.2 Open-source DSP candidates

| Need | Candidate | Notes |
|---|---|---|
| Amp modeling (HD tier) | Neural Amp Modeler core, RTNeural, or in-house small LSTM/GRU models | Core libraries MIT/BSD; each capture has its own license |
| Cabinet simulation | Partitioned FFT convolution (`fft-convolver`, `rustfft`, `realfft`) | Use permissively licensed IRs only |
| DSP graph and filters | `fundsp`, `dasp`, `biquad` | MIT/Apache |
| Time and modulation effects | Faust-generated Rust or in-house (Dattorro plate, FDN reverb, tape delay) | Verify library licenses |
| Drive and fuzz | Oversampled waveshapers; wave digital filter circuit models | Re-implement techniques; avoid copying GPL code |
| Pitch and harmony | `signalsmith-stretch`, PSOLA, phase vocoder, `pitch-detection` | Avoid GPL Rubber Band |
| Vocal noise reduction | `nnnoiseless` | BSD |
| Resampling | `rubato` | MIT |

License decision required early: GPL references (Guitarix, BYOD) are fine to study but copying code makes the app GPL.

### 6.3 Keeping it lightweight
- Release profile: fat LTO, one codegen unit, panic abort, stripped symbols; ship arm64-v8a (armeabi-v7a only if needed).
- Download amps, IRs and extra content on demand; bundle only a small starter set.
- Default to classic DSP amps; offer neural models only as an optional tier with a CPU check.
- Denormals off, oversampling only where aliasing is audible, performance hints to keep the audio thread on a suitable core.
- Fully offline, no account.

## 7. Platform constraints and risks

| Risk | Detail | Mitigation |
|---|---|---|
| Bluetooth input | Bluetooth mics use a narrow-band call profile and add high latency; Bluetooth output typically adds well over 100 ms | No live Bluetooth input; offer reamp mode, wired monitoring, latency indicator |
| Headset-jack latency and noise | Varies widely by phone; passive guitars sound thin without a preamp | Calibration, gain boost, in-app setup guidance |
| Neural amp CPU load | Standard NAM models may be too heavy for low-end phones | Two-tier amps, small models, device CPU check |
| Device fragmentation | Audio latency differs across vendors | Reference test devices, per-device profiles |
| License contamination | GPL code could force the app license | Decide license up front; track every dependency and asset license |
| AI art and API keys | User keys must be protected | Store in Android Keystore, send only to the chosen provider, local caching |
| Content licensing | Amp captures and IRs carry individual licenses | Curate a vetted catalog with license metadata |

## 8. Milestones and correctness gates

1. **Loopback gate:** input to output on analog iRig path and built-in mic, round-trip measurement, no dropouts over 10 minutes on 2-3 cheap phones.
2. **Engine gate:** gate, compressor, overdrive, delay, reverb and EQ working, verified offline against known test signals.
3. **Amp gate:** one DSP amp plus cabinet IR, CPU per block measured on the lowest reference device.
4. **Rig and Collection UI.**
5. **Pedal Maker** with schema, save/load and import/export.
6. **Instruments and learning tools:** voicing profiles, tuner, looper, metronome.
7. **BYOK cover art** and optional AI preset suggestion.
8. **HD amp tier** and downloadable content.

Each milestone ends only when its gate passes on the reference devices.

## 9. Open questions

- App license: GPL or permissive?
- Minimum Android version and lowest-supported device class?
- Which reference test phones (at least one low-end)?
- Source of curated IRs and amp models, and their redistribution terms?
- Content distribution: bundled starter set size, hosting for downloads?
- Monetization, if any?
- Which AI image providers to support in v1 (OpenAI-compatible endpoint first)?
- How ToneBridge compares on current features (to be researched before positioning is finalized)?