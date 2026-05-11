---
name: audio-plugin-ui-ux
description: Use when designing, implementing, or reviewing an audio plugin's user interface — knob layout, chassis composition, meter readability, signal-flow visualization, modulation halos, preset/A-B UI, DPI scaling, hardware-metaphor aesthetics, or pre-commit visual verification of plugin GUIs. Triggers on plugin UI mockups, knob/fader/segment work, plugin GUI builds in any framework (egui, JUCE, iPlug, Web Audio), and house-style decisions for Hyperfocus DSP / REXIST plugins.
version: 1.0.0
---

# Audio Plugin UI/UX

## Core principle

Audio plugins are real-time, performance-critical, mouse-and-keyboard instruments living inside someone else's host. They are not websites. Defaults imported from web/mobile UX (large hit areas, hover-reveal everything, generous whitespace, animation everywhere) frequently make plugins worse. Reason from first principles for this medium.

## When to use

- Designing or modifying a plugin's GUI in any framework
- Laying out knobs, faders, meters, segment displays, chassis
- Choosing a typography / color / skeuomorphism stance
- Reviewing UI work before committing
- Implementing presets, A/B compare, modulation visualisation, DPI scaling
- Establishing or honouring the developer's house style (Hyperfocus DSP / REXIST)

## Mandatory workflow

1. **Read `guidelines.md` in this skill directory** before designing or modifying any plugin UI.
2. **Cite specific rules** (`P1.3`, `L2.4`, `C4.1`, `H5.2`, `R7.4`) when justifying decisions in commits, PR descriptions, or your responses to the user.
3. **Run the §8 pre-completion checklist** in `guidelines.md` before claiming UI work is done.
4. **Surface conflicts** between `guidelines.md` and the current task to the user before acting. Never silently override.

## Hard rules (excerpt — full list in §7 of `guidelines.md`)

- Never move a user-approved knob row without explicit request.
- Never auto-shrink text to fit — fix the layout instead.
- Never claim UI work is done without running and visually verifying.
- Never mix skeuomorphic and flat controls in the same plugin.
- Never use pure `#FFFFFF` as the brand highlight; use bone `#F4F1EA`.
- Never invent or alter plugin names, parameter names, brand marks, or version numbers.
- Never overload a modifier key silently (must be discoverable via tooltip or docs).
- Never use linear frequency axes for spectrum displays — log only.
- Always render values with units, no bare numbers.
- Always provide both double-click reset *and* Cmd/Ctrl-click reset.
- Always integer-scale baked assets across DPI; don't re-flow at different scales.
- Always group with proximity before bounding boxes.

## Red flags — STOP and reconsider

- "I'll just move this knob row a little."
- "The contrast looks fine on my monitor."
- "I'll verify visually after I commit."
- "This NEVER-rule doesn't really apply here because..."
- "Type-checking passed, so the UI is done."
- "Pure white is close enough to bone."

Each of these means: stop, re-read `guidelines.md`, and reconsider with a cited rule.

## Quick reference

| Question | Section |
|---|---|
| Where does Norman/Nielsen apply to plugins? | §1 First principles |
| What's the minimum knob hit area? | §2 Cognitive laws (Fitts) |
| How do I group controls? | §3 Visual composition (Gestalt) |
| What's the knob interaction grammar? | §4 Audio conventions (C4.1) |
| What's the house palette / typography? | §5 House style |
| What process discipline applies to my workflow? | §6 Process |
| What must never happen? | §7 Hard rules |
| What do I check before saying done? | §8 Checklist |

## Full reference

→ `guidelines.md` in this skill directory.
