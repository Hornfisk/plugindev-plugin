# plugindev — Claude Code skill for nih-plug audio plugin development

Battle-tested patterns for building audio plugins with [nih-plug](https://github.com/robbert-vdh/nih-plug) and egui in Rust. Distilled from shipping two commercial plugins.

## What it covers

- Real-time safety (`assert_process_allocs`, `permit_alloc` discipline)
- Audio/GUI sync (atomics, try_lock, DisplaySnapshot pattern)
- DAW persistence (persist race gotcha and fix)
- Host transport (standalone vs DAW, `host_ever_stopped`)
- Voice pool (pre-allocation, age-based stealing, constant-power pan)
- Elektron-style sequencer (p-locks, pattern queueing, swing)
- Parameters (gain skew, smoothing, knob readback)
- egui editor (aspect-lock, double-click workaround, state versioning)
- Master bus DSP (compressor, saturator, limiter — RT-safe)
- Macro-knob compressor surface (3 musical knobs instead of 5 technical ones)
- Metering (atomic f32 round-trip, visual smoothing)
- Multi-stage distortion design (distinct-by-construction matrix)
- Testing (Goertzel fundamental extraction, drive=0 bypass, frequency response)
- Build, install, and DAW-specific gotchas (Renoise, PipeWire)

## Install

### Claude Code CLI

```bash
claude --plugin-dir /path/to/plugindev-plugin
```

Or clone into your plugins directory and reference from settings.

### From GitHub

```bash
git clone https://github.com/YOUR_USERNAME/plugindev-plugin.git
claude --plugin-dir ./plugindev-plugin
```

## Structure

```
plugindev-plugin/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   └── plugindev/
│       └── SKILL.md          # Main skill (320 lines)
├── references/
│   └── patterns.md           # Deep-dive patterns (237 lines)
├── LICENSE
└── README.md
```

## License

MIT
