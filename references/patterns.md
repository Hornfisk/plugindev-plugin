# nih-plug Pattern Reference

Deeper detail on patterns summarized in the main skill. Read the relevant section when you need implementation-level specifics.

## Table of Contents
1. [Audio/GUI Sync — full implementation](#audiogui-sync)
2. [Persist race — detailed sequence](#persist-race)
3. [Voice pool — complete implementation](#voice-pool)
4. [Sequencer — Elektron-style p-locks](#sequencer)
5. [SPSC queues for large handoffs](#spsc-queues)

---

## Audio/GUI Sync

### The SequencerSync pattern

A shared struct for hot sequencer state, accessed lock-free from both audio and GUI threads:

```rust
pub struct SequencerSync {
    pub current_step: AtomicUsize,
    pub is_playing: AtomicBool,
    pub tempo_x10: AtomicU32,       // BPM * 10 as integer (e.g. 1200 = 120.0 BPM)
    pub trigger_count: AtomicU32,   // monotonic, GUI uses delta to flash pads
    pub pattern_index: AtomicU8,
}
```

Wrap in `Arc`, clone one for the GUI. Audio thread does relaxed stores, GUI does relaxed loads. No lock contention, no allocation.

### The DisplaySnapshot pattern

Never hold a lock during egui layout. Instead:

```rust
struct DisplaySnapshot {
    step: usize,
    playing: bool,
    waveforms: Vec<Vec<f32>>,  // copied, not referenced
    kit_names: Vec<String>,
    // ... everything the GUI needs for one frame
}

// In the update() function:
let snapshot = {
    let guard = shared_state.lock();  // brief lock
    DisplaySnapshot {
        step: sync.current_step.load(Relaxed),
        waveforms: guard.waveforms.clone(),
        kit_names: guard.kit_names.clone(),
        // ...
    }
    // guard dropped here
};
// Now render from `snapshot` — no lock held
```

### try_lock discipline

Audio thread must never block:

```rust
if let Some(guard) = shared_state.try_lock() {
    // Process MIDI, update sequencer state, etc.
    permit_alloc(|| drop(guard));
} else {
    // Contended — skip this buffer's MIDI/seq updates.
    // A few ms gap is inaudible vs. blocking the RT thread.
}
```

---

## Persist Race

### The dangerous sequence

1. DAW loads project -> nih-plug deserializes `#[persist]` fields
2. Your plugin's `initialize()` runs, kicks off async work (library scan, preset load)
3. **Before async work completes**, a persist timer fires
4. Timer serializes current live state -> **defaults, not the restored DAW state**
5. DAW auto-saves -> your defaults overwrite the user's saved state

### The fix

```rust
struct MyPlugin {
    state_restored: bool,  // false by default
    persist_dirty: Arc<AtomicBool>,
    // ...
}

// In your async completion handler:
self.state_restored = true;

// In your persist timer:
if !self.state_restored { return; }
if !self.persist_dirty.swap(false, Ordering::Relaxed) { return; }
// Now safe to serialize
```

---

## Voice Pool

### Pre-allocation

```rust
const MAX_VOICES: usize = 32;

struct VoicePool {
    voices: Vec<Voice>,
    trigger_counter: u64,  // monotonic
}

impl VoicePool {
    fn new() -> Self {
        Self {
            voices: (0..MAX_VOICES).map(|_| Voice::default()).collect(),
            trigger_counter: 0,
        }
    }
}
```

### Voice stealing

```rust
fn trigger(&mut self, pad: usize, velocity: f32, sample: &Arc<Vec<f32>>) {
    self.trigger_counter += 1;

    // Find free voice, or steal oldest
    let voice = self.voices.iter_mut()
        .find(|v| !v.active)
        .unwrap_or_else(|| {
            self.voices.iter_mut()
                .min_by_key(|v| v.age)
                .unwrap()
        });

    // Fade out if still playing (retrigger on same pad)
    if voice.active && voice.pad == pad {
        voice.fade_out_samples = (0.05 * sample_rate) as usize;  // 50ms
    }

    voice.active = true;
    voice.age = self.trigger_counter;
    voice.pad = pad;
    voice.position = 0.0;
    voice.velocity = velocity;
    // Cache pan/pitch at trigger time
    voice.pan_angle = /* resolved from kit */;
    voice.pitch_ratio = /* resolved from kit */;
    permit_alloc(|| {
        voice.sample = Some(Arc::clone(sample));
    });
}
```

### Constant-power pan

```rust
let angle = pan * 0.25 * std::f32::consts::PI;  // pan: -1..1
let gain_l = (0.25 * PI - angle).cos();
let gain_r = (0.25 * PI + angle).sin();
// Equivalent: at pan=0, both channels get ~0.707 (-3dB)
```

---

## Sequencer

### P-lock (parameter lock) pattern

Optional per-step overrides that inherit from the pad default when None:

```rust
struct Step {
    enabled: bool,
    velocity: f32,
    probability: f32,       // 0.0..1.0
    pan: Option<f32>,       // None = use pad default
    pitch: Option<f32>,     // None = use pad default
    condition: ConditionTrig,
}

enum ConditionTrig {
    Always,
    Every(u8),      // fire every Nth time
    NotEvery(u8),   // fire all but every Nth
    Fill,           // only when fill button held
    NotFill,        // only when fill not held
}
```

### Pattern queueing

```rust
struct PatternBank {
    patterns: [Pattern; 16],
    active: usize,
    queued: Option<usize>,  // swap at next bar boundary
}

// In sequencer advance, when step wraps to 0:
if let Some(next) = self.queued.take() {
    self.active = next;
}
```

---

## SPSC Queues

For handing off large data (scanned sample library, loaded preset, waveform data) from a background thread to the audio thread without blocking:

```rust
use crossbeam_channel::{bounded, Receiver, Sender};

// Setup (in new()):
let (tx, rx): (Sender<LibraryData>, Receiver<LibraryData>) = bounded(1);

// Background thread:
tx.send(scanned_library).ok();

// Audio thread process():
if let Ok(library) = self.rx.try_recv() {
    permit_alloc(|| {
        self.library = library;
    });
}
```

`bounded(1)` ensures at most one pending update. `try_recv()` never blocks. The `permit_alloc` wraps any allocation that might happen when assigning the new data (e.g. dropping the old `Vec`).

For single-scalar feedback (not large data), prefer `AtomicU32` + `f32::to_bits/from_bits` — zero overhead, no channel needed.
