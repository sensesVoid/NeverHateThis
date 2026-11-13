# NeverHateThis — Design

Status: Draft for approval · Date: 2026-11-13 · Supersedes nothing · Related: [PRD v0.2](../PRD.md)

## 1. Intent

NeverHateThis is an offline-first Android app that turns a phone into a playable guitar, bass and vocal rig using the gear people already own: an iRig-style analog cable, a USB interface, or the built-in mic.

The core promise, in the owner's words: **use the phone connected to a normal speaker, or the phone itself, to make musical or vocal sound with effects.** It is a pocket amp/PA, not a headphone-only practice toy and not a song-preset catalog.

Why this can exist now: the incumbents treat Android as a compromise. ToneBridge's own store text supports only headphone-port adapters, and its engine vendor is sunsetting. A small, honest, open engine that works out loud on cheap hardware is a real gap.

## 2. Users and the core promise

| Persona | What they get |
|---|---|
| Budget beginner | Plug in, pick a tone recipe, hear it through the phone speaker; guided setup and forgiving defaults |
| Hobbyist guitarist/bassist | Amp + cab + pedalboard, tuner, metronome, looper; works with the cable in their drawer |
| Singer / home recorder | External mic → USB interface → phone → wired speaker/PA; phone is only the effect bridge |
| Tinkerer | Pedal Maker: combine blocks, choose knobs, design cover art, export a file and share it |

The product is judged by one moment: **plug in, and within seconds a sound comes out that makes you want to keep playing.**

## 3. Goals, non-goals, anti-goals

### Goals
- Live, playable effects on common Android phones, with measured latency shown honestly.
- Works with analog iRig-style inputs, USB audio interfaces (including mic inputs), and the built-in mic.
- Small install, low CPU, fully offline, no account.
- Pedal Maker with safe, shareable, data-only pedals.
- Open-source DSP stack with per-asset license tracking.
- Free, with donations as the only funding mechanism.

### Non-goals (v1)
- Live playing over Bluetooth input or output.
- Claiming a specific brand's tone; voicing profiles are labeled as input voicing.
- Cloud accounts, social network, marketplace, or upload server.
- Plugin hosting (VST/CLAP/LV2/AU).
- A 9000-preset song catalog. We do not compete on catalog size.

### Anti-goals
- An app that sounds fine in a demo but drops out on a real phone.
- A paywall, ad, subscription, or account of any kind.
- A file format that can execute code or fetch unverified remote content.
- Feature parity with ToneBridge at the cost of the speaker/live use case.

## 4. Constraints and prior art

### 4.1 What ToneBridge actually is (public evidence)
- Android engine: **Superpowered C++ SDK** ("This app has been Superpowered", app store text). Commercial, closed source, **sunsetting**: no new licenses, operations end 2027-07-31; a successor (Yuby) exists from the same founder.
- macOS build: **"powered by AmpKit"**, a different third-party engine. Ultimate Guitar licenses engines per platform; their in-house work is content and UX.
- Android I/O: **headphone-port adapters only**; USB is explicitly unsupported. Reviews praise headphone/amp output and report major delay through a speaker.
- Content: 9000+ song presets, 7500+ demo clips, roughly 29 amps / 30 pedals / 26 cabs / 8 mics, Workshop, publish + rating. ~20.7 MB APK; catalog is server-side.
- Their published latency model: best-case device round trip ~35 ms, with the app-side ring buffer the largest controllable slice (~10 ms).

Implication: ToneBridge is a catalog play wearing an amp app. Its moat is content, not DSP, and its weak points are exactly our core: speaker output, USB, vocals, cheap hardware.

### 4.2 Platform constraints
- AAudio exists from API 26, but Google does not recommend it on Android 8.0 (a stream-close crash bug; Oboe deliberately falls back to OpenSL ES there). Oboe uses AAudio by default from API 27, and the MMAP low-latency path is enabled from Android 8.1. Our floor is API 29, which matches the 2020+ device class.
- USB audio host support is OEM-dependent; some interfaces work, some don't. Must be tested per device, never promised globally.
- Bluetooth adds >100 ms. Live playback over Bluetooth is out.
- Phone speaker + built-in mic monitoring is an acoustic feedback loop. For vocals the phone is only an effect bridge: external mic in, wired speaker out.
- AudioFlinger, the HAL and the driver add fixed latency we do not control; we can only minimize our own buffers, resampling and processing.

## 5. Decisions (closed)

| # | Question | Decision |
|---|---|---|
| 1 | App license | **Apache-2.0** for code; **CC BY 4.0** for starter presets; per-file license metadata for every IR/model. GPLv3 rejected because we do not reuse GPL code and want app-store and commercial options open. |
| 2 | Android floor | **API 29 (Android 10), arm64-v8a only**, matching the 2020+ device class. x86_64 debug only. |
| 3 | Reference devices | Three **2020-or-newer** phones: gate device (Redmi 9A / Galaxy A12 class, 2–3 GB), mid (Pixel 4a / Galaxy A54), high (Pixel 8). Plus iRig 2 and one USB interface with a real mic input. |
| 4 | IR / amp sources | Only content we make or can prove: our own algorithmic amps, our own measured or synthetic IRs, user-imported captures never bundled without written permission. |
| 5 | Distribution | **GitHub Releases only** — a signed arm64 APK per `v*` tag, plus content packs on Releases behind an Ed25519-signed manifest with SHA-256 addressing. No Play account, no AAB, no backend. |
| 6 | Monetization | **Free, all the way.** Donations only, no unlocks, no ads, no subscription, no account. |
| 7 | AI providers | **BYOK**: OpenAI, Gemini, Anthropic, OpenRouter. Provider capabilities are declared per task. Keys stay in Android Keystore. |
| 8 | ToneBridge comparison | Closed (section 4.1). Compete on live/speaker, USB, vocals, user pedals, honesty. Do not compete on catalog. |

Two decisions are expensive to reverse: the license (1) and the API floor (2). The rest are cheap corrections.

## 6. Approaches considered

| | Approach | Trade-off | Verdict |
|---|---|---|---|
| A | **Sound-first spike**: throwaway Oboe C++ + Kotlin shell, input → gate → drive → cab IR → output, calibration, CPU meter, three phones | Fastest truth; ~300 lines discarded | **Chosen, first** |
| B | **Rust engine first**: real workspace, DSP crates, chain, params, UniFFI, Compose | Nothing wasted, but weeks before a guitar is audible and device risk unmeasured | **Chosen, second** |
| C | Product-first: Rig/Collection/Maker against a stub engine | Demos fast; project-killing risk untouched; rework likely | Rejected |

A exists to produce the numbers B needs: buffer sizes, CPU headroom, USB behavior, device quirks. Keeping the spike's measurement harness is allowed; keeping its UI or DSP is not.

## 7. What v1 includes

- Rig: ordered chain (input → pedals → amp → cab → output); add/remove/reorder/bypass; knobs; save/load; click-free swaps; meters; default noise gate.
- Amp: three algorithmic voicings (clean, crunch, lead) built from a preamp waveshaper, passive tone stack, power-amp stage and a parametric cabinet; no captures and no neural models.
- Pedals: gate, compressor, overdrive, distortion, fuzz, EQ, chorus, flanger, phaser, tremolo, delay, reverb, wah; de-esser and monophonic pitch correction for vocals.
- Pedal Maker: combine blocks, expose chosen parameters as knobs, versioned JSON, cover art (templates, image import, BYOK AI generation), export/import.
- Input: instrument modes (vocal, electric guitar, bass), voicing profiles, gain/DC block/HPF/boost, loopback calibration, feedback guard.
- Tools: tuner, metronome, looper, backing-track player, tone recipes, signal-chain explainer, record-dry-then-reamp.
- Community: export/import `.nhtpedal` files, Discord link for chat, GitHub Discussions for sharing and archiving. No marketplace, no accounts.
- Donations: About screen links only, no feature gating.

## 8. Architecture

### 8.1 Overview

```
Kotlin + Jetpack Compose
  ├── screens, state, navigation, permissions, file pickers
  ├── Android Keystore (AI keys), share sheet, downloads
  └── UniFFI control API
        ↕ commands / snapshots / meters
Rust workspace
  ├── nht-engine   chain graph, parameter bus, crossfades, meters
  ├── nht-dsp      blocks, filters, oversampling, convolution
  ├── nht-content  schemas, manifest, license checks, verification
  └── nht-audio    Oboe integration, calibration, device profiles
        ↕ C ABI, no JNI on the audio thread
Oboe → AAudio (MMAP where available) → audio device
```

UniFFI covers the control plane only. The audio callback lives entirely in Rust (or in a thin C++ Oboe shim that calls one Rust `process` function). Default: **C++ Oboe shim (~150 lines) plus a C ABI**, because it isolates Oboe's C++ callback lifetime from the Rust engine and keeps the real-time path obvious. `oboe-rs` is the fallback if the shim fights the build.

### 8.2 Threading and real-time contract

- Audio thread: no allocation, no locks, no I/O, no logging, no JNI, no `String`.
- Parameter changes: atomics per parameter, one-pole smoothing (5–20 ms, per parameter).
- Structural changes (add/remove/reorder): built on a worker thread, swapped in via an atomic pointer, crossfaded 10–20 ms. Old chain freed on the worker thread after a grace period.
- Meters: audio thread writes `AtomicU32` (f32 bits); UI reads at 30 Hz.
- Debug builds install an allocator that aborts if the audio thread allocates.
- Denormals flushed; oversampling only where aliasing is audible (drive/fuzz).

### 8.3 Chain and parameter model

- Blocks implement `process(&mut self, block: &mut AudioBlock, params: &Params)` plus `reset()`.
- Parameters are a fixed `[AtomicU32; N]` per block, addressed by stable numeric IDs from the schema. No string lookups on the audio thread.
- The UI owns the patch document; Rust owns only real-time state. Loading a patch pushes a full parameter set, then the chain.
- Every block declares: parameter ranges, defaults, units, taper (linear/log), and whether it is mono or stereo.
- v1 is strictly serial: input → mono pedals → amp → cab → output. No parallel paths, no FX loops, and the cab is always last. The cab is the only mono→stereo point; v1 output is dual-mono. The block trait carries a channel count from day one so a stereo post-cab section can land in v1.1 without a preset-format migration.

### 8.4 Audio I/O and latency

- 48 kHz, f32, mono in, stereo out.
- Oboe stream: `InputPreset::Unprocessed`, low-latency performance mode, exclusive where available.
- Input routing: one picker driven by `AudioManager.getDevices(GET_DEVICES_INPUTS)` plus an L/R channel toggle for 2-input interfaces. No per-OEM routing UI. USB is promised only for the reference interface that passes calibration.
- Buffer: 192 frames default, negotiated down/up by calibration. Low-end gate: 256 frames, zero xruns over 10 minutes.
- App-side contribution target: ≤12 ms at 192 frames (input ring + processing + output ring).
- Calibration: loopback cable test (output → input) for true round trip, plus Oboe timestamp estimate. Stored per device + route. Shown in Settings and on first run.
- Latency is displayed, never hidden. No "low latency" claim without a number.

### 8.5 DSP blocks

- All f32, in place, no allocation. Biquads from a shared coefficient library; state reset on swap.
- Drive/fuzz: input HPF, gain staging, 4× oversampled asymmetric waveshaper, tone network, output trim.
- Modulation: delay-line based chorus/flanger/phaser, LFO smoothed.
- Delay: tape-style with LP in the feedback path, tempo sync optional.
- Reverb: Dattorro plate plus an FDN variant; no GPL code.
- Cabinet: parametric IIR model by default (speaker resonance, inductance shelf, low-pass and mic distance), with an optional FIR/IR convolution path (partitioned FFT, 256-frame blocks, 200 ms max) for later licensed or user IRs. Mono → stereo out.
- Vocal: de-esser as band-split compressor; pitch correction monophonic only (pitch detection + resampling), labeled as such.
- Tuner: YIN, ±1 cent, A4 reference configurable.

### 8.6 Amp and cab

- Signal chain per voicing: input HPF → preamp (asymmetric waveshaper or a Koren/Leach triode stage) → passive tone stack (Yeh's Bassman/Marshall/Vox equations) → power amp (sag, presence, output-transformer low-pass) → cabinet.
- **No captures, no neural models, no NAM import.** The owner has no amps to record, so the strategy is circuit-informed DSP tuned by ear against reference recordings. The reference set is in `docs/research/amp-voicing-references.md`.
- Cabinet is parametric by default: speaker resonance (fs, Qts), inductance high-shelf, low-pass and mic distance. An optional FIR/IR path exists for later licensed or user IRs; none ship in v1.
- Three voicings ship: clean (Fender-ish), crunch (Marshall-ish), lead (hot-rodded). Names are descriptive, never brand claims.

### 8.7 UI

- Single Activity, Compose, Material 3. Screens: Rig, Collection, Pedal detail, Maker, Tools, Settings, About.
- Rig is a horizontal chain with drag-to-reorder; pedal detail is a knob panel; Maker is blocks → knobs → art.
- Visual identity: high-contrast, readable outdoors, large touch targets, no skeuomorphic clutter.
- Onboarding: input detection (which route is live), calibration, then a "make a sound" recipe.
- v1 keeps audio in the foreground with the screen on (`FLAG_KEEP_SCREEN_ON`). Screen-off playback via a microphone foreground service is v1.1: it adds permissions, a notification, audio-focus handling and Play review work, none of which the core play-through-the-room moment needs.

### 8.8 Persistence and formats

- `.nhtrig` (rig/preset) and `.nhtpedal` (pedal) are versioned JSON with `schema: 1`.
- Pedals contain only declarative block IDs, parameter values, knob mappings and art references. **No code, no scripts, no remote URL dependencies.**
- Unknown required fields reject the file; unknown optional fields are preserved on round-trip.
- Assets are referenced by SHA-256 plus license/attribution, never by a bare URL.
- Library: files in app storage with a JSON index; no database until it earns its place.
- Keys and imported files never enter cloud backup (`allowBackup=false` for key material).

### 8.9 Content pipeline

- Starter pack ≤4 MB: 3 amps, 8 pedals, 3 cabs, ~12 presets.
- Remote packs: static files + `manifest.json` (Ed25519 signature, per-file SHA-256, license, attribution, size).
- Client verifies signature and hashes before install; rejected files are deleted.
- APK target ≤15 MB, total install ≤20 MB.

### 8.10 AI (BYOK)

- One provider interface with capability flags: `image`, `text`.
- Adapters: OpenAI (image + text), Gemini (image + text), Anthropic (text only), OpenRouter (depends on model), plus a custom OpenAI-compatible base URL.
- Cover art UI offers only providers that declare `image`; text→pedal offers all.
- Keys: per-provider entries in Android Keystore, never logged, redacted in errors, one-tap clear.
- Requests go directly from the device to the provider. No proxy, no telemetry.
- The non-AI path (templates, palettes, image import) is always available and never worse.
- Text→preset is v1.1; the schema validator ships in v1 so the path is safe when it lands.

### 8.11 Community and donations

- Sharing is file-based: export via the Android share sheet; import validates before anything is added.
- Discord invite for chat; **GitHub Discussions** as the searchable archive for pedal files (Discord attachments are not an archive).
- No marketplace, no upload server, no account, no ratings backend.
- Donations: GitHub Sponsors / Ko-fi / Liberapay links on the About screen. Donations unlock nothing; they are pure support.

### 8.12 Distribution and CI

- **GitHub Releases is the only channel.** A `v*` tag builds a signed arm64 APK plus `SHA256SUMS` and release notes. No Play account, no AAB, no F-Droid in v1.
- Users install with "install unknown apps"; updates come from Releases, optionally tracked by Obtainium. The SHA-256 check is documented in the release notes.
- CI: GitHub Actions runs Rust fmt/clippy/tests and the Android debug build on every push and PR. The release job runs on tags, decodes the keystore from repository secrets, builds `assembleRelease` and publishes the APK.
- The signing key lives offline with the owner; only the base64 keystore and its passwords live in GitHub Actions secrets.
- Versioning: SemVer tags, `versionCode` derived from the tag, release notes as the changelog.

## 9. Testing and gates

### Offline correctness
- Golden tests per block against reference renders (SciPy/NumPy), tolerance-based.
- Bypass null test; gain-staging test; aliasing sweep for drive blocks; IR convolution accuracy test.
- Schema fuzz tests: malformed, oversized, cyclic and hostile pedal files.

### Real-time
- 10-minute xrun test per device and route, logged via Oboe xrun counts.
- CPU per block and per default chain, measured, not estimated.
- Allocation guard armed in all debug runs.

### Device gates
- The 2020-or-newer gate device must pass 10 minutes at 256 frames with the default chain.
- USB round trip ≤20 ms is promised only on mid/high devices that pass calibration.
- Any device that fails is shown as unsupported with an honest reason, plus offline/reamp mode.

## 10. Milestones and rough sizes

Solo estimates, to be reset by the spike.

| Stage | Work | Size |
|---|---|---|
| M0 (A) | Spike: Oboe + gate + drive + cab sim + calibration + CPU meter on three 2020+ phones | 1–2 weeks |
| M1 (B) | Rust engine core: blocks, chain, params, crossfade, golden tests | 4–6 weeks |
| M2 | Amp voicings + parametric cab, CPU measurement | 3–4 weeks |
| M3 | Rig + Collection UI | 4–6 weeks |
| M4 | Pedal Maker + import/export + community flow | 3–4 weeks |
| M5 | Tools: tuner, metronome, looper, backing track, reamp | 3–4 weeks |
| M6 | AI cover art + donations + About | 1–2 weeks |
| M7 | Content packs + voicing refinement | 4–6 weeks |

Total: roughly 6–8 months of solo work. M0 can kill or reshape everything after it, which is the point.

## 11. What's left to the builder

- Exact DSP topologies, tone-stack values and gain staging per amp.
- Whether the Oboe shim or `oboe-rs` wins in practice (default: shim).
- Voicing recipes and tone-stack component values per amp; the reference list is the starting point.
- Visual design of the Rig and Maker screens.
- Whether the backing track is mixed in Rust or played by ExoPlayer beside the chain (default: Rust mixing for one output stream).
- Looper length, storage and undo model.
- Splash of marketing voice; the brand is the owner's call.

## 12. Risks

| Risk | Mitigation |
|---|---|
| Cheap phone cannot reach playable latency | M0 measures before we build; honest device floor; offline/reamp mode |
| USB host support varies by OEM | Test matrix; publish supported interfaces; analog fallback |
| Acoustic feedback with speaker output | Vocals are bridged through an external mic and wired speaker; feedback guard + notch on guitar |
| Solo bandwidth over 6–8 months | Milestones are gates; ship M3 as a usable practice amp if later stages slip |
| Content licensing | Parametric cab and in-house voicings mean no third-party audio assets in v1; any future IRs carry per-file metadata and a signed manifest |
| Mimicking amps without captures may not satisfy golden-ear players | Reference A/B renders and iterative tone-stack tuning; honest labeling as voicings, not emulations |
| Rust/Android toolchain friction | Spike uses C++/Oboe to keep device truth independent of Rust setup |

## Appendix: sources

- ToneBridge store listing and changelog (package `com.ultimateguitar.tonebridge`), 2026.
- Ultimate Guitar, "Free Tool to Create Your Own Tonebridge Presets" (29 amps / 30 pedals / 26 cabs / 8 mics), 2018.
- Superpowered, "Superpowered SDK: The Farewell Tour" (deprecation, 2027-07-31), and "Android Audio's 10 Millisecond Problem" (latency budget), accessed 2026-11-13.
- MuseHub, Tonebridge macOS listing ("powered by AmpKit"), accessed 2026-11-13.
- Google Play ToneBridge description (headphone-port adapters only), accessed 2026-11-13.
