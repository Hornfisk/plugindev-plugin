---
name: plugindev
description: Use whenever working on nih-plug / Rust audio plugins or audio-DSP Rust projects — VST3/CLAP, egui plugin UI, voice pool, sequencer, or real-time audio thread code. Triggers on mentions of nih-plug, VST3, CLAP, egui editor, audio thread, permit_alloc, assert_process_allocs, process() buffer, voice stealing, host transport, DAW persistence, or plugin bundling. Encodes proven patterns so new plugin work avoids re-inventing RT safety, audio/GUI sync, persistence, and bundling every time.
tools: Read, Write, Edit, Bash, Glob, Grep
version: 1.0.0
---

# Plugin Development (nih-plug + egui)

## Prime directive

If your project already has a working plugin codebase, grep it before designing any new subsystem:

```bash
rg -l <topic> src/
```

Copy existing idioms and adapt names. This saves re-deriving RT-safety, thread sync, and persistence patterns.

For deeper reference on any topic below, see `references/patterns.md` in this skill's directory.

## Real-time safety (audio thread)

nih-plug's `assert_process_allocs` panics on any heap allocation during `process()`. Keep the feature on — it catches real bugs. Wrap known false positives:

```rust
use nih_plug::util::permit_alloc;

// parking_lot mutex drop (unlock_slow may allocate)
permit_alloc(|| drop(shared));

// Arc clone/drop
permit_alloc(|| { voice.sample = Some(Arc::clone(sample)); });
permit_alloc(|| { self.data = None; });  // Arc::drop

// thread spawn, channel bounded, HashSet insert — all need permit_alloc
```

**Rule of thumb:** any op touching an allocator (Arc refcount, Vec growth, HashMap, spawn, formatted tracing) must be wrapped. Plain scalar/atomic ops are free.

## Audio <-> GUI sync (three layers)

1. **Hot state -> atomics.** Wrap in `Arc<SyncStruct>` shared with GUI. Use `AtomicUsize`/`AtomicBool`/`AtomicU8` for step pos, playing, trigger counts, tempo (BPM*10 as u32). Lock-free both directions.

2. **Bulk state -> `Arc<parking_lot::Mutex<SharedState>>`.** Audio thread uses `try_lock()`, never `lock()`. If contended, skip MIDI/sequencer for this buffer — a few ms gap is inaudible vs blocking the RT thread. Always `permit_alloc(|| drop(guard))`.

3. **GUI render -> DisplaySnapshot.** Per frame: brief lock, copy metadata + waveforms into a plain struct, drop the lock, render from snapshot. No lock held during egui layout.

Handoff of large results (scanned library, loaded preset) -> `crossbeam-channel::bounded(1)`, `try_recv()` in process().

## DAW persistence

```rust
#[persist = "plugin-state"]
pub plugin_state: Arc<parking_lot::Mutex<String>>,  // JSON blob
```

**Persist race gotcha:** if you have async init (background scan, preset load), nih-plug deserializes the persist field before `initialize()` runs, but your async work hasn't restored it into live state yet. A periodic persist timer firing in that window will overwrite the DAW's saved state with defaults.

**Fix:** `state_restored: bool` flag, default false. Gate the persist timer on it. Set true only after the async work has applied any DAW-restored state. Audio thread sets a `persist_dirty` atomic; GUI thread does the actual `serde_json::to_string` (so the audio thread never allocates).

## Host transport: standalone vs DAW

Standalone backends (CPAL, PipeWire/JACK) report `transport.playing = true` forever. Real DAWs toggle it.

```rust
host_ever_stopped: bool,  // set true on any buffer where !transport.playing
// host_driving = self.host_ever_stopped && transport.playing && !internal_play
```

Expose an "internal play" toggle for standalone. Only follow host transport when `host_ever_stopped` is true.

**Incremental beats:** accumulate beats (not samples) in internal play mode so tempo changes don't jump position. When following host, derive current step from `transport.pos_beats()` directly — no drift accumulation.

## Voice pool (sample playback)

- Fixed `MAX_VOICES`, `Vec<Voice>` pre-allocated.
- Age-based voice stealing: monotonic `trigger_counter`, steal lowest `age`.
- Retrigger: fade out existing voice on the same pad (50ms) instead of hard-cut.
- Cache resolved pan/pitch at trigger time so `process()` doesn't need the kit ref.
- Linear interpolation for pitch; constant-power pan via `cos/sin(angle)`.
- Sample data is `Arc<Vec<f32>>` — shared between library and voices. Drop the Arc inside `permit_alloc`.

## Sequencer (Elektron-style)

- `Pattern { lanes: Vec<Lane>, swing: f32 }`, `Lane { steps: [Step; 16], muted, solo }`.
- `Step { enabled, velocity, probability, pan: Option<f32>, pitch: Option<f32>, condition }` — `Option` fields are p-locks (None = inherit pad).
- `ConditionTrig`: Always, Every(N), NotEvery(N), Fill, NotFill.
- `PatternBank { patterns: [Pattern; 16], active, queued }` — queued swap happens at bar boundary (step 0).
- Swing: even steps lengthened, odd shortened, `swing * base * 0.5` offset.

## Parameters (nih-plug Params)

```rust
master_volume: FloatParam::new(
    "Master Volume",
    util::db_to_gain(0.0),
    FloatRange::Skewed {
        min: util::db_to_gain(-60.0),
        max: util::db_to_gain(6.0),
        factor: FloatRange::gain_skew_factor(-60.0, 6.0),
    },
)
.with_smoother(SmoothingStyle::Logarithmic(10.0))
.with_unit(" dB")
.with_value_to_string(formatters::v2s_f32_gain_to_db(2))
.with_string_to_value(formatters::s2v_f32_gain_to_db()),
```

- **Gain params**: `Skewed` range with `gain_skew_factor`, `Logarithmic` smoother.
- **Linear params**: `Linear` range + `Linear` smoother (ms).
- Pull smoothed values once per sample in the master loop, not once per buffer — cheaper and click-free:
  ```rust
  for i in 0..num_samples {
      let gain = self.params.master_volume.smoothed.next();
      // ...
  }
  ```

## egui editor (nih_plug_egui)

- Create with `create_egui_editor(editor_state, user_state, build, update)`.
- `EguiState::from_size(w, h)` — serialized via `#[persist = "editor-state-vN"]`. Bump the version key when layout changes incompatibly so old saved sizes don't break new builds.
- **Knob readback:** `param.unmodulated_plain_value()`, not `value()`. `value()` includes host modulation and makes the knob jitter.
- **Knob writes:** `begin_set_parameter` -> `set_parameter_normalized` on drag -> `end_set_parameter` on release.
- **Double-click:** baseview doesn't fire `response.double_clicked()`. Track last click time per widget; delta < 0.4s = double. Store in ui state.
- **Aspect-lock scaling:** set `pixels_per_point` from window size to lock content aspect ratio.

## Master bus DSP (RT-safe)

- Pre-allocate all buffers in `new()`, never `Box::new`/`Vec::with_capacity` during `process()`.
- `prepare(sample_rate)` called from `Plugin::initialize()` **and** `Plugin::reset()` — recompute envelope coefficients: `coeff = exp(-1 / (ms * 0.001 * sr))`. Reset is important so DAW play/stop/seek doesn't carry envelope history across a transport jump.
- Canonical chain: RMS compressor -> tanh saturator -> brickwall limiter. Store state (env_db, rms_sum, rms_buf, lim_env) as plain fields.
- One per-sample loop for the whole chain + volume is cheaper and cleaner than iterating the buffer multiple times.
- **Dirty-check expensive coefficient recomputes.** If attack/release are user-exposed params, `set_times(atk_ms, rel_ms, sr)` should only call `exp()` when the values actually change — otherwise you're computing 48k exps/sec for nothing. Store `last_atk_ms` / `last_rel_ms` as f32 fields.
- **Bypass branch pays off.** At the call site, skip the whole `process_sample` when all dynamics params are at their "clean" defaults (`amount < eps && drive < eps && !limiter_on`). Makes the feature bit-identical to a pre-feature build when disabled, protects existing presets, and costs one branch.

## Macro-knob compressor surface

For instrument plugins where the user wants *glue*, not pro-level control, collapse the classic 5-knob comp into 3 musical macros:

- **AMOUNT** drives `threshold` down and `ratio` up simultaneously: `threshold_db = lerp(-6, -30, amount)`, `ratio = lerp(2, 10, amount)`. Auto-makeup already handled inside the DSP core.
- **REACT** drives attack and release together in opposite directions: `attack_ms = lerp(30, 1.5, react)` (slow->snappy), `release_ms = lerp(400, 40, react)`.
- **DRIVE** is passed straight to the post-compressor tanh stage (0..1, `if drive > 0.001` gate inside the DSP).
- **LIM** is a BoolParam toggling the brickwall limiter at a fixed ceiling (e.g. -0.3 dBFS).

All four mappings are pure scalar arithmetic, computed per-sample from smoothed params in the chain loop — no branches, no allocations. Plus an outer bypass branch when everything is at defaults.

## Metering (audio -> GUI) for dynamics feedback

For GR meters, output meters, or any single-scalar-per-buffer feedback:

```rust
// util/telemetry.rs
pub struct MeterShared { gr_db_bits: AtomicU32 }
impl MeterShared {
    pub fn store_gr_db(&self, v: f32) { self.bits.store(v.to_bits(), Relaxed); }
    pub fn load_gr_db(&self) -> f32 { f32::from_bits(self.bits.load(Relaxed)) }
}
```

- Audio thread: one `store` per *buffer* (not per sample) with the max value over that buffer. Single Relaxed atomic store, allocator-free.
- GUI thread: one `load` per frame, then a one-pole visual smoother (instant attack, ~180 ms release via `exp(-dt / 0.18)`) so the meter doesn't flicker when audio buffers are quiet.
- `AtomicU32` + `f32::to_bits`/`from_bits` is the cheapest way to round-trip an `f32` through a relaxed atomic when the value is guaranteed non-negative.

## Visualising post-FX signal in an existing waveform display

If you already have a telemetry ring feeding a waveform renderer, and you add a new master-bus effect (compressor, limiter, saturation), **move the telemetry capture point to *after* the new effect**. The existing renderer then automatically reflects the post-FX signal with zero code changes — the compressed waveform visibly shrinks, the limited peaks visibly cap, etc.

Net cost: one line moved. Net benefit: the visual feedback users want, for free.

## Cargo.toml skeleton

```toml
[lib]
crate-type = ["cdylib"]

[[bin]]
name = "<plugin>-standalone"
path = "src/main.rs"

[dependencies]
nih_plug = { git = "https://github.com/robbert-vdh/nih-plug", features = ["assert_process_allocs", "standalone"] }
nih_plug_egui = { git = "https://github.com/robbert-vdh/nih-plug" }
parking_lot = "0.12"
crossbeam-channel = "0.5"
rtrb = "0.3"               # for SPSC audio param queues
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["fmt", "env-filter"] }

[profile.release]
lto = "thin"
strip = "symbols"
```

Export:

```rust
nih_export_vst3!(plugin::MyPlugin);
nih_export_clap!(plugin::MyPlugin);
```

Implement `ClapPlugin` + `Vst3Plugin` with unique CLAP ID and 16-byte VST3 class ID.

## Logging

Write a small logging module that writes to `$XDG_DATA_HOME/<plugin>/<plugin>.log`, truncates over 500KB keeping last 5000 lines, initialized once via `std::sync::Once`. Tail with `tail -f ~/.local/share/<plugin>/<plugin>.log`.

## Config & preset paths

Platform-aware:
- **Linux:** `$XDG_CONFIG_HOME/<plugin>/config.json` (config), `$XDG_DATA_HOME/<plugin>/presets/` (presets)
- **macOS:** `~/Library/Application Support/<plugin>/`
- **Windows:** `%APPDATA%/<plugin>/`

Preset struct should include a `version: u32` field and `#[serde(default)]` on new optional fields for forward-compat.

## Build & install

```bash
# Dev build + cargo xtask bundle
cargo xtask bundle <plugin> --release

# Install VST3 (rm -rf first!)
rm -rf ~/.vst3/<Plugin>.vst3
cp -r target/bundled/<Plugin>.vst3 ~/.vst3/

# Install CLAP
cp -f target/bundled/<plugin>.clap ~/.clap/
```

**Critical:** always `rm -rf` the target `.vst3` directory before `cp -r`. Otherwise `cp -r src dest` nests bundles (`dest/X.vst3/X.vst3/...`) and hosts silently fail to scan.

In Bitwig: right-click plugin browser -> Rescan Plugins. In Renoise: Preferences -> Plug/Misc -> Rescan.

## DAW-specific gotchas

### Renoise: self-sequencing plugins route to master

Renoise routes a plugin instrument's audio to whichever track has an active note-on for that instrument. When a plugin's internal sequencer generates audio without a Renoise note trigger, there's no track claim — audio falls through to the master bus.

**Workaround:** Place a sustained note (e.g. C-4, no note-off) on line 00 of the target track. This claims the instrument on that track so internal sequencer output routes through the correct DSP chain. Re-trigger on line 00 if the note gets cut by a pattern break. This is standard Renoise behavior for any self-sequencing plugin, not a nih-plug bug.

### PipeWire: standalone routing

nih-plug standalones built with `--backend alsa` go through: cpal ALSA host -> `default` PCM -> pipewire-alsa shim -> PipeWire default sink. They do NOT follow per-stream routes that browsers may have set explicitly.

If a standalone plugin is silent but browser audio works, the first diagnostic is `wpctl status` — check where the `*` default sink marker is vs. where the browser streams are actually routed. If they differ, `wpctl set-default <sink-id>` fixes it.

## Testing patterns

- **Unit-test the voice pool** with known sample data (e.g., `vec![1.0; 8]`) and assert on buffer output with expected constant-power pan gain (`(0.25 * PI).cos()` ~= 0.7071).
- **Unit-test sequencer** without a real VoicePool — fake trigger_flags via `[AtomicU8; N]` and assert step-fire counts/order.
- **Integration smoke-test** via standalone: `cargo run --release --bin <plugin>-standalone` and verify logs.

### Frequency-response tests through saturating/shaping stages

**Never use wideband RMS** to test how a nonlinear stage treats a specific frequency. A saturating shaper fed a sine produces the fundamental *plus* a harmonic stack — wideband RMS lumps them together, and a heavily saturated LF sine can appear "louder" than the input because its square-ish waveform contains enormous HF harmonic energy.

Measure the **fundamental-bin power only**, via a single-bin Goertzel-style correlation:

```rust
fn fundamental_power(samples: &[f32], sr: f32, bin_freq: f32) -> f32 {
    let w = std::f32::consts::TAU * bin_freq / sr;
    let (mut re, mut im) = (0.0f32, 0.0f32);
    for (i, &x) in samples.iter().enumerate() {
        let p = w * i as f32;
        re += x * p.cos();
        im += x * p.sin();
    }
    re * re + im * im
}
```

O(N), no FFT dependency, ignores all harmonics. Use for: HP/LP bypass tests, drive-reactive tape HF loss, transformer-vs-tanh harmonic signature comparisons, EQ interacting with drive.

### Drive=0 must be an explicit bypass

If a shaper is nonlinear at unity gain (rational `x/sqrt(1+x^2)`, cubic `x - x^3/3`, split-band clippers, etc.), "drive=0 -> clean passthrough" is NOT an emergent property of the math. Add an explicit early-return when drive is below ~1e-4:

```rust
if mode == SatMode::Off || mix <= 0.0 || drive <= 1e-4 {
    return input;
}
```

### Distinct-by-construction matrix for multi-stage distortion

When a plugin has multiple distortion stages, users will rightly complain that "they all sound the same" if two or more are the same `tanh`-family curve. `tanh`, `x/(1+|x|)`, `x/sqrt(1+x^2)`, and `1.5x - 0.5x^3` all share a symmetric-odd-harmonic signature.

Enforce **distinct-by-construction**: each stage must occupy a different cell in the (symmetry x harmonic weighting x dynamics x frequency dependence) grid:

| Stage | Symmetry | Harmonics | Dynamic? | Freq-dep? |
|---|---|---|---|---|
| Tube warmth | asym (bias) | 2nd-dom cubic | no | no |
| Split-band clip | symmetric | odd hard | no | yes (HP sub bypass) |
| Diode pair | very asym | even+odd | no | no |
| Tape | mild asym | 2nd+3rd smeared | yes (hysteresis) | yes (drive-reactive LP) |
| Transformer | symmetric | 2nd+3rd poly | no | yes (LF bloom) |

No two stages should overlap on all four axes. If you find yourself reaching for another tanh variant, pick from the non-tanh options instead.

## Debug checklist

1. **Plugin not loading in DAW?** Check rescan; check `ldd` on `.so` for missing deps; confirm no nested bundle (`ls ~/.vst3/<Plugin>.vst3` should show `Contents/`, not another `.vst3`).
2. **Audio dropouts / xruns?** Run with `RUST_LOG=<plugin>=debug`; look for `parking_lot unlock_slow` or allocator panics. Add `permit_alloc` around the offending line.
3. **State lost on DAW reload?** Likely the persist race — check `state_restored` gating. Also check `#[persist = ...]` key hasn't changed.
4. **Sequencer drifts from host?** Confirm you're deriving current step from `pos_beats` directly, not accumulating samples.
5. **Knobs jitter under automation?** Switch from `value()` to `unmodulated_plain_value()`.
6. **Standalone auto-plays?** Missing `host_ever_stopped` guard.
7. **Plugin audio goes to DAW master instead of track?** If using Renoise with a self-sequencing plugin, place a sustained note on the target track (see DAW gotchas above).
8. **Standalone silent but browser audio works?** Check PipeWire default sink with `wpctl status` — standalone follows the default, browsers may have per-stream routes.

## When starting a new subsystem

1. Grep your existing codebase — is there an existing pattern?
2. If not, design it, then document the result before moving on.
3. Prefer small per-file unit tests on the pure-DSP logic; the full plugin can be smoke-tested via standalone.
