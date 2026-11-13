# M0 Sound Spike Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a throwaway Android measurement app that proves whether 2020+ phones with cheap inputs can run a representative guitar chain at playable latency, and produce the numbers that shape the Rust engine.

**Architecture:** Kotlin/Compose shell over a C++ Oboe full-duplex engine: input callback → SPSC ring buffer → output callback → gate/drive/parametric cab. Stats are atomics read by the UI at 10 Hz; measurements are written to CSV in app storage. No Rust, no product UI, no content, no AI.

**Tech Stack:** Kotlin 2.0.21, Compose BOM 2024.12.01, AGP 8.7.3, Gradle 8.9, NDK 27.2.12479018, Oboe 1.9.0, C++20, CMake 3.22, JUnit 4.13.2.

**Spec:** `docs/design/2026-11-13-nht-design.md` (§7, §8.4, §9)

**Scope:** This plan covers M0 only. M1 (the Rust engine) is planned after M0's numbers land, because the design says M0 resets the estimates. Everything here is throwaway except the measurement harness and the report.

## Global Constraints

- `minSdk 29`, `compileSdk 35`, `targetSdk 35`.
- Release ABI: `arm64-v8a` only. Debug ABI: `arm64-v8a` and `x86_64`.
- 48 kHz, f32, mono input, stereo output.
- Audio thread: no allocation, locks, file I/O, logging or JNI.
- Default burst 192 frames; calibration candidates 96, 128, 192, 256, 384, 512.
- Low-end gate: 256 frames, zero xruns over 10 minutes.
- App-side latency target: ≤12 ms at 192 frames.
- Every measurement is keyed by device model and input route.
- Throwaway: no Rust, no product screens, no persistence beyond calibration.
- License: Apache-2.0. DSP files that follow a paper carry a source comment.

## Review Focus

- Input device removed or changed mid-stream: engine restarts or stops cleanly, with no frozen UI and no noise burst. Tested in Task 4.
- Device refuses the requested burst and returns a larger one: calibration records `getFramesPerBurst()`, never the requested value. Tested in Task 4.
- Silent or denied input: no NaN, no runaway gain, and a clear "no input" state. Tested in Task 3.
- Instrument on input 2 of a USB interface: the channel toggle reaches it, and opening mono does not fail. Tested in Task 4.
- High-gain phone-speaker output: feedback guard and warning, and the report records it. Tested in Tasks 3 and 6.

---

### Task 1: Oboe passthrough skeleton

**Files:**
- Create: `android/settings.gradle.kts`, `android/build.gradle.kts`, `android/gradle.properties`, `android/gradle/wrapper/gradle-wrapper.properties`, `android/gradlew`, `android/gradlew.bat`, `android/app/build.gradle.kts`, `android/app/src/main/AndroidManifest.xml`
- Create: `android/app/src/main/java/dev/nht/spike/MainActivity.kt`, `SpikeEngine.kt`
- Create: `android/app/src/main/cpp/CMakeLists.txt`, `jni_bridge.cpp`, `audio_engine.h`, `audio_engine.cpp`, `ring_buffer.h`
- Create: `android/app/src/main/res/values/strings.xml`, `themes.xml`
- Test: device check on the gate phone (no test file)

**Interfaces:**
- Consumes: nothing.
- Produces: `SpikeEngine.start(sampleRate, framesPerBurst, inputDeviceId, inputChannels, selectedChannel): Boolean`, `SpikeEngine.stop()`, and the C++ `AudioEngine` below.

```cpp
struct EngineConfig {
  int32_t sampleRate = 48000;
  int32_t framesPerBurst = 192;
  int32_t inputDeviceId = 0;    // 0 = default
  int32_t inputChannels = 1;    // request 2 for a stereo USB input
  int32_t selectedChannel = 0;  // 0 = L, 1 = R
};

class AudioEngine : public oboe::AudioStreamDataCallback {
 public:
  bool start(const EngineConfig& config);
  void stop();
  oboe::DataCallbackResult onAudioReady(
      oboe::AudioStream* stream, void* audioData, int32_t numFrames) override;
};
```

- [ ] **Step 1: Create the Gradle project**

Use `minSdk 29`, `compileSdk 35`, `targetSdk 35`, `ndkVersion = "27.2.12479018"`, `externalNativeBuild.cmake.path = "src/main/cpp/CMakeLists.txt"`, `buildFeatures.prefab = true`, and `buildFeatures.compose = true`. Dependencies: `com.google.oboe:oboe:1.9.0`, `androidx.activity:activity-compose:1.9.3`, `androidx.compose:compose-bom:2024.12.01`, `androidx.compose.material3:material3`, `androidx.compose.ui:ui-tooling-preview`, `testImplementation("junit:junit:4.13.2")`. Plugins: `com.android.application` 8.7.3, `org.jetbrains.kotlin.android` 2.0.21, `org.jetbrains.kotlin.plugin.compose` 2.0.21. Wrapper: Gradle 8.9. Manifest: `RECORD_AUDIO`, `MODIFY_AUDIO_SETTINGS`, `uses-feature android.hardware.microphone required="true"`, `application android:label="NHT Spike"`, `allowBackup="false"`.

- [ ] **Step 2: Add the lock-free ring buffer**

Create `ring_buffer.h` with `template <typename T, size_t Capacity> class SpscRingBuffer` exposing `bool write(const T*, size_t)`, `size_t read(T*, size_t)`, `size_t available() const`, `void clear()`. Capacity must be a power of two; use `std::atomic<size_t>` read/write indices and a mask. Instantiate `SpscRingBuffer<float, 8192>` for mono frames.

- [ ] **Step 3: Implement `AudioEngine::start`**

Open an input stream (`Direction::Input`, `InputPreset::Unprocessed`, `PerformanceMode::LowLatency`, `Format::Float`, 48 kHz, `inputChannels`, `setDeviceId`, `setDataCallback(this)`) and an output stream (`Direction::Output`, `PerformanceMode::LowLatency`, `Format::Float`, 48 kHz, 2 channels, `Usage::Game`, `setDataCallback(this)`). Try `SharingMode::Exclusive`, retry `Shared` on failure. Set `setFramesPerDataCallback(framesPerBurst)` on both; record the actual `getFramesPerBurst()` and `getSharingMode()`. Start output, then input; stop input, then output.

- [ ] **Step 4: Implement the callbacks**

In `onAudioReady`, branch on `stream == inputStream_.get()`. Input: for each frame, copy channel `selectedChannel` into the mono ring; if `write` fails, increment `overruns`. Output: `read` `numFrames`; zero-fill any shortfall and count `underruns`; copy mono to both output channels (temporary passthrough). Return `DataCallbackResult::Continue`. Do not allocate, log or lock in either callback.

- [ ] **Step 5: Add the JNI bridge and Kotlin object**

Create `SpikeEngine.kt` as `object SpikeEngine { init { System.loadLibrary("nhtspike") }; external fun start(...); external fun stop() }`. In `jni_bridge.cpp`, implement `Java_dev_nht_spike_SpikeEngine_start` and `_stop` with a `static std::unique_ptr<AudioEngine>`. Return `true` when `start` succeeds.

- [ ] **Step 6: Build the minimal UI**

`MainActivity` requests `RECORD_AUDIO`, shows a `Start`/`Stop` button and the labels `rate`, `burst (actual)`, `sharing mode`. Poll `Build.MODEL` and `AudioManager.getProperty(PROPERTY_OUTPUT_SAMPLE_RATE / PROPERTY_OUTPUT_FRAMES_PER_BUFFER)` for the device's preferred values; pass 48000/192 to `start`.

- [ ] **Step 7: Build and verify on the gate phone**

Run:
```bash
cd android && ./gradlew :app:assembleDebug
adb install -r app/build/outputs/apk/debug/app-debug.apk
adb logcat -s NhtSpike
```
Expected: `BUILD SUCCESSFUL`, install succeeds, and logcat prints `engine started rate=48000 requested=192 actual=<n> sharing=<exclusive|shared>`. With an iRig connected and headphones on, pressing Start gives clean passthrough. Record what happens when the input is unplugged; Task 4 makes it recover cleanly.

- [ ] **Step 8: Commit**

```bash
git add android
git commit -m "spike: oboe full-duplex passthrough skeleton"
```

---

### Task 2: Stats, latency estimate and CSV logging

**Files:**
- Modify: `android/app/src/main/cpp/audio_engine.h`, `audio_engine.cpp`, `jni_bridge.cpp`
- Modify: `android/app/src/main/java/dev/nht/spike/SpikeEngine.kt`, `MainActivity.kt`
- Create: `android/app/src/main/java/dev/nht/spike/EngineStats.kt`, `MeasurementLogger.kt`
- Test: device check plus a CSV file inspection

**Interfaces:**
- Consumes: `AudioEngine` from Task 1.
- Produces: `SpikeEngine.stats(): DoubleArray` and `MeasurementLogger`.

```kotlin
data class EngineStats(
  val xruns: Long, val callbacks: Long,
  val avgCallbackUs: Double, val maxCallbackUs: Double,
  val inputLatencyMs: Double, val outputLatencyMs: Double, val roundTripMs: Double,
  val cpuPercent: Double, val sampleRate: Int, val framesPerBurst: Int,
  val sharingMode: Int // 0 = shared, 1 = exclusive
)

class MeasurementLogger(context: Context) {
  fun start(): File
  fun append(stats: EngineStats, route: String)
  fun stop(): File
}
```

- [ ] **Step 1: Add the counters and latency estimate**

Add atomics to `AudioEngine`: `callbacks`, `callbackUsSum`, `callbackUsMax`, `ringUnderruns`, `ringOverruns`, `outputLatencyMs`. In the output callback, time the work with `clock_gettime(CLOCK_THREAD_CPUTIME_ID)` and update the atomics; compute `cpuPercent = (sum / callbacks) / (1e6 * framesPerBurst / sampleRate) * 100`. Estimate output latency from `outputStream_->getTimestamp(CLOCK_MONOTONIC)`: `nowMs - tsMs + (ts.framePosition - framesWritten) * 1000 / sampleRate`, clamped to ≥0. Poll `getXRunCount()` for both streams inside `stats()` and add the ring counters. `stats()` returns the `EngineStats` fields.

- [ ] **Step 2: Expose stats through JNI**

`SpikeEngine.stats()` returns an 11-element `DoubleArray` in this order: xruns, callbacks, avgCallbackUs, maxCallbackUs, inputLatencyMs, outputLatencyMs, roundTripMs, cpuPercent, sampleRate, framesPerBurst, sharingMode. `inputLatencyMs` and `roundTripMs` may be `-1.0` until Task 5; the UI must render them as `—`.

- [ ] **Step 3: Show the stats**

Map the array in `SpikeEngine.stats(): EngineStats`. In `MainActivity`, poll every 100 ms while running and render: device model, route name, rate, burst, sharing mode, xruns, avg/max callback µs, CPU %, and latency estimate. Keep the Start/Stop button enabled at all times.

- [ ] **Step 4: Write the CSV**

`MeasurementLogger` writes to `context.getExternalFilesDir(null)/m0-<model>-<yyyyMMdd-HHmmss>.csv` with the header `timestamp_ms,model,sdk,route,sample_rate,burst,sharing,xruns,callbacks,avg_cb_us,max_cb_us,latency_ms,cpu_pct`. `append` writes one row; flush every 10 rows and on `stop`. Start logging with Start and stop with Stop. No permissions are needed for the app-specific external directory.

- [ ] **Step 5: Verify and commit**

Run the app for 30 seconds with the iRig connected. Expected: stats update at 10 Hz, xruns stay 0, CPU is plausible (<10% passthrough). Pull the CSV:
```bash
adb shell ls /sdcard/Android/data/dev.nht.spike/files/
adb pull /sdcard/Android/data/dev.nht.spike/files/<file>.csv
```
Expected: header plus one row per 100 ms, monotonic `timestamp_ms`.

```bash
git add android
git commit -m "spike: stats, latency estimate and CSV logging"
```

---

### Task 3: Gate, drive and parametric cab

**Files:**
- Create: `android/app/src/main/cpp/dsp/biquad.h`, `biquad.cpp`
- Create: `android/app/src/main/cpp/dsp/gate.h`, `gate.cpp`
- Create: `android/app/src/main/cpp/dsp/drive.h`, `drive.cpp`
- Create: `android/app/src/main/cpp/dsp/cab.h`, `cab.cpp`
- Create: `android/host-tests/CMakeLists.txt`, `test_dsp.cpp`
- Modify: `android/app/src/main/cpp/audio_engine.cpp`, `MainActivity.kt`, `.github/workflows/ci.yml`
- Test: `android/host-tests/test_dsp.cpp`

**Interfaces:**
- Consumes: `AudioEngine` from Tasks 1–2.
- Produces:

```cpp
struct GateParams { float thresholdDb = -60.0f; float attackMs = 2.0f; float releaseMs = 150.0f; };
struct CabParams {
  float resonanceHz = 85.0f, resonanceQ = 1.2f, resonanceGainDb = 3.0f;
  float shelfHz = 1250.0f, shelfGainDb = 3.0f;
  float lowPassHz = 5500.0f;
};

class Biquad {
 public:
  void setLowPass(double fs, double f0, double q);
  void setHighPass(double fs, double f0, double q);
  void setPeaking(double fs, double f0, double q, double gainDb);
  void setHighShelf(double fs, double f0, double q, double gainDb);
  void reset();
  float process(float x);
};

class Gate  { public: void prepare(double fs); void reset(); void setParams(const GateParams&); void process(float*, int32_t); };
class Drive { public: void prepare(double fs); void reset(); void setGainDb(float); void process(float*, int32_t); };
class Cab   { public: void prepare(double fs, const CabParams&); void reset(); void process(float*, int32_t); };
```

- [ ] **Step 1: Write the failing host tests**

Create `android/host-tests/test_dsp.cpp` with a `CHECK(cond)` macro that prints `FAIL file:line cond` and calls `std::exit(1)` on failure, and these cases at 48 kHz. Include `<algorithm>`, `<cmath>`, `<cstdio>` and the four `dsp/*.h` headers.

```cpp
bool biquad_dc_gain() {
  Biquad b; b.setLowPass(48000, 1000, 0.707); b.reset();
  float y = 0; for (int i = 0; i < 48000; ++i) y = b.process(1.0f);
  CHECK(std::abs(y - 1.0f) < 0.01f); return true;
}
bool gate_closes_silence() {
  Gate g; g.prepare(48000); g.setParams({-60, 2, 150}); g.reset();
  float buf[256] = {}; float peak = 0;
  for (int i = 0; i < 100; ++i) { g.process(buf, 256); for (float v : buf) peak = std::max(peak, std::abs(v)); }
  CHECK(peak < 1e-6f); return true;
}
bool drive_no_nan() {
  Drive d; d.prepare(48000); d.setGainDb(40); d.reset();
  float buf[256]; float peak = 0;
  for (int i = 0; i < 100; ++i) {
    for (int n = 0; n < 256; ++n) buf[n] = 0.5f * std::sin(2 * M_PI * 1000 * (i * 256 + n) / 48000.0);
    d.process(buf, 256);
    for (float v : buf) { CHECK(std::isfinite(v)); peak = std::max(peak, std::abs(v)); }
  }
  CHECK(peak < 10.0f); return true;
}
bool cab_stable() {
  Cab c; c.prepare(48000, {}); c.reset();
  float buf[256] = {}; buf[0] = 1.0f; c.process(buf, 256);
  float tail = 0;
  for (int i = 0; i < 100; ++i) { float z[256] = {}; c.process(z, 256); for (float v : z) tail = std::max(tail, std::abs(v)); }
  CHECK(std::isfinite(tail)); CHECK(tail < 0.01f); return true;
}
```

- [ ] **Step 2: Run the tests to see them fail**

```bash
cmake -S android/host-tests -B build/host-tests -DCMAKE_BUILD_TYPE=Release
cmake --build build/host-tests -j
ctest --test-dir build/host-tests --output-on-failure
```
Expected: the build fails with missing `dsp/*.h`.

- [ ] **Step 3: Implement the four DSP files**

`Biquad` uses the RBJ cookbook and transposed direct form II with `double` coefficients and state; `reset()` zeroes the state. `Gate` is a peak envelope follower with threshold, attack/release and a smoothed gain (no clicks). `Drive` is an 80 Hz high-pass, gain stage, 4× oversampled `tanh` with a 2nd-order low-pass at 20 kHz before and after, and an output trim. `Cab` is the peaking resonance, high shelf and two cascaded low-pass biquads from `CabParams`. All process in place and allocate nothing.

- [ ] **Step 4: Run the tests to see them pass**

Run: `ctest --test-dir build/host-tests --output-on-failure`
Expected: `100% tests passed, 0 tests failed out of 1`. If `cab_stable` fails, lower `resonanceQ` or check the shelf gain sign before touching the test.

- [ ] **Step 5: Wire the chain into the engine**

In the output callback, after reading the mono ring: `gate.process` → `drive.process` → `cab.process`, unless bypass is on. Add `SpikeEngine.setBypass`, `setDriveDb` and `setCabEnabled` (atomics read once per callback). UI: a bypass switch, a drive dB slider (-12…40) and a cab switch.

- [ ] **Step 6: Run the host tests in CI**

In `.github/workflows/ci.yml`, add this step to the `android` job before the Gradle build:

```yaml
      - name: Host DSP tests
        run: |
          cmake -S android/host-tests -B build/host-tests -DCMAKE_BUILD_TYPE=Release
          cmake --build build/host-tests -j
          ctest --test-dir build/host-tests --output-on-failure
```

- [ ] **Step 7: Verify on the device and commit**

With the iRig connected: bypass should null to passthrough; drive at 0 dB should be close to bypass; 30 dB should distort; cab on should remove the fizz. CPU at 30 dB drive with cab must stay under 25% on the gate phone. Unplugging the guitar must not produce NaN or a stuck gate.

```bash
git add android .github
git commit -m "spike: gate, drive and parametric cab with host tests"
```

---

### Task 4: Buffer calibration and route handling

**Files:**
- Create: `android/app/src/test/java/dev/nht/spike/CalibrationTest.kt`
- Create: `android/app/src/main/java/dev/nht/spike/Calibration.kt`, `BufferCalibrator.kt`, `AudioRouteMonitor.kt`
- Modify: `MainActivity.kt`, `SpikeEngine.kt`
- Test: `CalibrationTest.kt`

**Interfaces:**
- Consumes: `EngineStats`, `SpikeEngine.start/stop/stats` from Tasks 1–3.
- Produces:

```kotlin
fun pickBurst(tested: List<Pair<Int, Int>>): Int?
data class CalibrationResult(val burst: Int, val tested: List<Pair<Int, Int>>)

class BufferCalibrator(
  private val stop: () -> Unit,
  private val restart: (requestedBurst: Int) -> Unit,
  private val stats: () -> EngineStats,
) { suspend fun run(secondsPerStep: Int = 15): CalibrationResult }

class AudioRouteMonitor(context: Context, onChanged: (String) -> Unit)
```

`pickBurst` returns the smallest requested burst whose xrun count is 0, or `null` if none qualify. `BufferCalibrator` tries 96, 128, 192, 256, 384 and 512 in that order, restarting at each and reading `stats().xruns` after `secondsPerStep`.

- [ ] **Step 1: Write the failing unit test**

Create `CalibrationTest.kt`:

```kotlin
class CalibrationTest {
  @Test fun empty_returns_null() = assertNull(pickBurst(emptyList()))
  @Test fun picks_smallest_zero_xrun() =
    assertEquals(192, pickBurst(listOf(96 to 4, 128 to 1, 192 to 0, 256 to 0)) ?: -1)
  @Test fun skips_xruns() =
    assertEquals(256, pickBurst(listOf(128 to 2, 192 to 1, 256 to 0)) ?: -1)
  @Test fun none_zero_returns_null() =
    assertNull(pickBurst(listOf(96 to 1, 128 to 1)))
  @Test fun unsorted_input_is_sorted() =
    assertEquals(128, pickBurst(listOf(256 to 0, 128 to 0, 192 to 0)) ?: -1)
}
```

- [ ] **Step 2: Run it to see it fail**

Run: `cd android && ./gradlew :app:testDebugUnitTest --tests '*CalibrationTest'`
Expected: FAIL — `pickBurst` unresolved.

- [ ] **Step 3: Implement `pickBurst`**

In `Calibration.kt`, `pickBurst` sorts by requested burst and returns the first pair with `second == 0`, else `null`. Record the **actual** burst from `stats().framesPerBurst` in `CalibrationResult`, never the requested value.

- [ ] **Step 4: Run it to see it pass**

Run: `./gradlew :app:testDebugUnitTest --tests '*CalibrationTest'`
Expected: `BUILD SUCCESSFUL`, 5 tests passed.

- [ ] **Step 5: Integrate calibration**

`BufferCalibrator.run` calls `stop()`, then `restart(candidate)` for each burst, waits `secondsPerStep`, reads `stats()`, and stops. Pick with `pickBurst`; if `null`, report "no stable buffer" and fall back to 512. Persist `CalibrationResult` in `SharedPreferences` (`calibration_burst`, `calibration_json`). On launch, start with the stored burst (default 192). UI: a `Calibrate` button that shows a progress row per candidate and the final actual burst.

- [ ] **Step 6: Handle routes and channels**

`AudioRouteMonitor` registers an `AudioDeviceCallback` and emits a human route label (`built-in mic`, `headset`, `USB: <productName>`). On any add/remove, stop and restart the engine and refresh the UI. Add an input-device dropdown from `AudioManager.getDevices(GET_DEVICES_INPUTS)` and an L/R toggle; pass `inputChannels = 2` when a USB device is selected and `1` otherwise. Selecting input 2 must not fail when the HAL exposes one channel — fall back to channel 0 and show a note.

- [ ] **Step 7: Verify on the gate phone**

Run Calibrate with the iRig connected. Expected: a burst is chosen, persisted, and survives an app restart. Then unplug the headset mid-stream, replug it, and attach/detach a USB interface. Expected: the stream restarts, the UI updates the route, no crash, no stuck tone. With a 2-input interface, select L and R and confirm the signal moves.

- [ ] **Step 8: Commit**

```bash
git add android
git commit -m "spike: buffer calibration and audio route handling"
```

---

### Task 5: Loopback round-trip measurement

**Files:**
- Create: `android/app/src/main/cpp/measure.h`, `measure.cpp`
- Create: `android/host-tests/test_measure.cpp` (add it to `host-tests/CMakeLists.txt`)
- Modify: `audio_engine.h`, `audio_engine.cpp`, `jni_bridge.cpp`, `SpikeEngine.kt`, `MainActivity.kt`, `MeasurementLogger.kt`
- Test: `android/host-tests/test_measure.cpp`

**Interfaces:**
- Consumes: `AudioEngine` from Tasks 1–4.
- Produces:

```cpp
struct LoopbackResult { bool detected; double latencyMs; };
LoopbackResult detectLoopback(const float* recorded, int32_t frames,
                              double sampleRate, int32_t clickOffsetFrames);
```

- [ ] **Step 1: Write the failing test**

At 48 kHz, allocate 48,000 zeros. Put a 5-sample click of `1.0f` at index 480 and a copy of `0.5f` at index 480 + 2400. Expect `detected == true` and `latencyMs` within 0.1 ms of `50.0`.

- [ ] **Step 2: Run it to see it fail**

```bash
cmake --build build/host-tests -j
ctest --test-dir build/host-tests --output-on-failure
```
Expected: link error — `detectLoopback` undefined.

- [ ] **Step 3: Implement `detectLoopback`**

Scan `recorded` from `clickOffsetFrames + 0.01 * sampleRate` to the end; track the sample with the largest absolute value. If that peak is below `0.1f`, return `{false, -1.0}`. Otherwise return `{true, (peakIndex - clickOffsetFrames) * 1000.0 / sampleRate}`. Ignore the first 10 ms so the direct click cannot be mistaken for the return.

- [ ] **Step 4: Run it to see it pass**

Run: `ctest --test-dir build/host-tests --output-on-failure`
Expected: both tests pass.

- [ ] **Step 5: Wire the loopback mode**

Add `SpikeEngine.startLoopback()` / `stopLoopback()`. In loopback mode the output plays a 5 ms 1 kHz click every 500 ms and the chain is bypassed. The input callback copies one second of samples into a preallocated 48,000-float buffer (no allocation in the callback). When the buffer is full, call `detectLoopback` on a worker thread, store the result, and expose it through `stats()` as `roundTripMs`. Add a `Loopback` button and show the result with the route name.

- [ ] **Step 6: Verify on the gate phone**

Built-in mic and speaker: press Loopback, stay quiet, expect a plausible number (usually 25–60 ms) that is stable within ±3 ms over five runs. With the iRig and a cable from output to input: repeat and record. With USB: repeat if the interface allows a cable loop; otherwise record "timestamp estimate only" in the report. Save every result to CSV.

- [ ] **Step 7: Commit**

```bash
git add android
git commit -m "spike: loopback round-trip measurement"
```

---

### Task 6: Ten-minute soak and the M0 report

**Files:**
- Create: `tools/m0_report.py`
- Create: `docs/m0-report.md` (output of the script, edited with conclusions)
- Modify: `docs/design/2026-11-13-nht-design.md` (§9 device gates, link to the report)
- Test: the soak runs themselves

**Interfaces:**
- Consumes: CSV files from Task 2, loopback results from Task 5.
- Produces: `python tools/m0_report.py <csv-dir>` → Markdown on stdout.

- [ ] **Step 1: Run the soak on each device and route**

For every combination, start the app, press Calibrate, then run for 10 minutes with the default chain (gate + drive at 15 dB + cab). Routes: iRig analog, USB interface, built-in mic (vocal). Devices: gate phone, mid phone, high phone. Export each CSV. Expected: one CSV per combination, named `<model>-<route>-<timestamp>.csv`.

- [ ] **Step 2: Write the aggregator**

`tools/m0_report.py` reads every CSV in the directory (stdlib `csv` only), groups by `model` + `route`, and emits a Markdown table with: device, SDK, route, burst, sharing, round-trip ms, xruns per 10 min, avg CPU %, max CPU %, verdict. It must tolerate missing rows and print `n/a` rather than crash.

- [ ] **Step 3: Generate and annotate the report**

Run: `python tools/m0_report.py <csv-dir> > docs/m0-report.md`
Add: the exact device models and Android versions, the USB interface model, the loopback method (acoustic vs cable), and a short note on feedback for phone-speaker output.

- [ ] **Step 4: Apply the go/no-go gate**

The gate passes only if all three hold:
1. Gate phone: zero xruns for 10 minutes at ≤256 frames.
2. Gate phone: average CPU under 25% for the default chain.
3. Mid/high phone over USB: measured round trip ≤20 ms.

If the gate fails, record the failing number and the smallest change that would pass (larger burst, smaller chain, no oversampling on that device). Do not soften the target.

- [ ] **Step 5: Link the result from the design**

In `docs/design/2026-11-13-nht-design.md` §9, add one line: `M0 measured results: [docs/m0-report.md](../m0-report.md)`. If the gate failed, add one sentence naming the consequence for M1.

- [ ] **Step 6: Commit and push**

```bash
git add tools docs
git commit -m "spike: M0 soak report and go/no-go"
git push origin main
```

## Self-Review

- Spec coverage: M0 gate (loopback, 10-minute xruns, CPU, devices) is Tasks 5–6; latency/route rules are Tasks 2 and 4; the DSP chain is Task 3.
- Type consistency: `EngineStats` fields, `start` parameters and the `stats()` array order are defined once and reused.
- Review focus: each line names the task that tests it.
- Proportion: six tasks, each with a buildable deliverable; no DSP body is transcribed.

