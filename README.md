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

In Claude Code, add this repo as a plugin marketplace and install the plugin:

```
/plugin marketplace add Hornfisk/plugindev-plugin
/plugin install plugindev@hornfisk-plugins
```

To update later:

```
/plugin marketplace update hornfisk-plugins
```

## Structure

```
plugindev-plugin/
├── .claude-plugin/
│   ├── marketplace.json      # Marketplace catalog
│   └── plugin.json           # Plugin manifest
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
