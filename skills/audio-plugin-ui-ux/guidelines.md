# Audio Plugin UI/UX Guidelines

> Operating manual for the developer (human or AI) building audio plugin interfaces.
> First principles → cognitive laws → visual composition → audio conventions → house style → process → hard rules → checklist.

---

## 0. How to read this document

**Audience.** An AI coding assistant (Claude, etc.) implementing or reviewing audio plugin UI code.

**Priority of instructions.** When two sources conflict, the higher wins:

1. The user's explicit instructions in the current session (CLAUDE.md, GEMINI.md, AGENTS.md, direct messages).
2. The hard rules in §7 of this document.
3. The principles, laws and conventions in §§1–6.
4. General best practice and your training.

**Reference syntax.** Use `P1.3` to cite Principle 1.3, `L2.4` for Law 2.4, `C4.1` for Convention 4.1, `H5.2` for House style 5.2, `R7.4` for Rule 7.4. Cite when justifying a decision.

**Disagreement protocol.** If you believe a rule in this document is wrong for the situation, **surface the conflict to the user before acting**. Never silently override. Never delete a rule because it inconveniences your current edit.

**Scope.** Stack-agnostic. No framework-specific code patterns (egui, JUCE, iPlug, Flutter, Web Audio). The principles apply to all; the *mechanics* are the developer's job.

**Mental model.** Audio plugins are real-time, performance-critical, mouse-and-keyboard instruments that live inside someone else's host. They are not websites. Defaults imported from web UX (large hit areas, hover-reveal everything, generous whitespace, animation everywhere) frequently make plugins worse. Reason from first principles for this medium.

---

## 1. First principles

Distilled from Norman (*The Design of Everyday Things*), Nielsen's 10 Usability Heuristics, Shneiderman's 8 Golden Rules, Tognazzini's First Principles. Each principle here is the *plugin-specific* application, not the textbook abstraction.

### P1.1 — Affordance and signifier

A control's appearance must match what it does. A knob looks turnable. A button looks pressable. A drop zone looks droppable. **Decorative controls are bugs.** If the user can't predict what an element does from looking at it, redesign it or remove it.

Corollary: do not invent new control types when an established one fits. A vertical slider that resets on double-click is a slider; do not call it a "fader strip" with novel interactions.

### P1.2 — Feedback within one audio frame

Every direct-manipulation action (drag, click, scroll) must produce visible response within ~16 ms (one frame at 60 Hz). The Doherty threshold (400 ms) is a *ceiling*, not a target — that's the threshold for "feels instant" for indirect actions. For knob drags, anything > 1 frame breaks the illusion of control.

Apply: drive the GUI redraw from input events as well as the timer. Never wait on audio-thread state for visual response if a parameter cache can give it.

### P1.3 — Mapping: control direction matches audible direction

Clockwise increases. Up increases. Left-to-right is "earlier in signal chain." A knob whose value rises while turning counter-clockwise is broken even if the label is correct. Bipolar parameters fill outward from center.

### P1.4 — Constraints and error prevention

Prevent invalid states. Clamp ranges. Disable controls that don't apply (route off → disable the post-route knobs). Refuse impossible drag targets visually. Recovery is more expensive than prevention.

### P1.5 — Conceptual model coherence

The plugin must teach the user its model in the first 30 seconds of looking at it. A kick synth that's actually a sample-mangler with a synth front-end will confuse forever, no matter how good the DSP is. Make the model visible: signal flow, what affects what, where modulation enters.

If the model is complex, **don't simplify by hiding** — simplify by grouping and labelling.

### P1.6 — Recognition over recall

Display the unit (Hz, dB, %, ms). Display the current value. Display the default position. Don't make the user remember "the second knob from the left in the third row is decay." Label every primary control. Tooltips augment labels; they don't replace them.

### P1.7 — Match between system and the real world

If you adopt a hardware metaphor (chassis, screws, LED display) **commit to it**. Half-skeuomorphism reads as confused. The metaphor must hold across every element: a "screw" next to a flat material-design toggle is a contradiction.

Hardware metaphor is not the only valid choice — Ableton-style flat / pixel-honest works. Pick a stance, stay in it.

### P1.8 — Close the gulfs of execution and evaluation

Norman's gulfs:

- **Gulf of execution** — "how do I make this do what I want?" Closed by affordances, labels, sane defaults, discoverable shortcuts.
- **Gulf of evaluation** — "did it do what I wanted?" Closed by metering, value displays, A/B compare, undo, audible feedback.

Every parameter must answer both gulfs. A knob with no value readout fails evaluation. A button with no state indicator fails execution.

---

## 2. Cognitive laws

These are predictive. They tell you *in advance* what will work.

### L2.1 — Fitts's Law: target acquisition

Time to acquire a target ∝ distance ÷ size. In practice:

| Role | Minimum hit area | Notes |
|---|---|---|
| Primary parameter (main knobs) | 40 px on shortest side @ 1× | Most-touched controls live here. |
| Secondary parameter | 24 px | Trim knobs, sub-settings. |
| Toggle / button | 24 × 24 px | Bigger if used during play. |
| Modifier zones (segments, dots) | 16 px | Acceptable for keyboard-anchored use. |
| Resize grip, edge controls | 12 px hit zone | Visual can be smaller; pad the hit zone. |

Pad the hit zone larger than the visual where possible. Never use the *visible* size as the hit zone for small controls.

### L2.2 — Hick's Law: choice cost

Decision time grows with number of equally-weighted choices. Apply:

- First-glance count of primary parameters ≤ 7. Above that, group or page.
- Don't surface power-user features at the same visual weight as everyday ones.
- A "more" / "advanced" reveal is acceptable when the hidden parameters are genuinely tertiary.

### L2.3 — Miller's Law: chunking

Working memory holds 5–9 items. Audio plugins exploit this: an envelope is one chunk (ADSR), a filter is one chunk (cutoff/res/type), an EQ band is one chunk. The user thinks in chunks; lay out in chunks.

Each chunk should be visually self-contained (proximity, optional bounding) and labelled at the chunk level, not just the parameter level.

### L2.4 — Jakob's Law: users expect your plugin to work like other plugins

The cost of breaking a convention is paid by every user, forever. Break a convention only when the new behavior is *significantly* better and you are willing to document the difference. Pre-approved deviations: zero.

See §4 for the conventions you must honour by default.

### L2.5 — Doherty threshold: response under 400 ms

For non-direct-manipulation actions (load preset, recalc IR, render preview), the user-perceived response must be < 400 ms or you must show progress. Anything in between (200–800 ms with no indicator) feels broken. Either be fast or be honest.

### L2.6 — Aesthetic-usability effect

A polished surface raises perceived quality and usability — *up to a point*. Beyond that point, polish is wasted budget. Apply:

- Ship a coherent, finished surface even if the feature set is small.
- Do **not** ship a beautiful UI over broken DSP or unclear behavior. Users forgive ugly faster than they forgive lying.

---

## 3. Visual composition

### V3.1 — Gestalt grouping by proximity

Things that belong together should be closer to each other than to things that don't. Whitespace groups before boxes do. Use bounding boxes only when proximity is insufficient (mixed adjacent groups, density constraints).

Order of preference for grouping: proximity > similarity (color/shape) > enclosure (box, panel) > common fate (animation, modulation).

### V3.2 — Typographic hierarchy

- **Numerics: monospaced** (or proportional with tabular figures enabled). Values must not jitter horizontally as they change.
- **Labels: sans-serif at the smallest legible size for the screen.** Baseline 10 pt at 1× scale.
- **Units rendered smaller than values**, same baseline.
- **Never auto-shrink text to fit** — fix the layout instead. Auto-shrink produces inconsistent visual weight and signals a bug.
- One body face, one display face maximum.
- Capitalize labels as their type warrants: PARAMETER LABELS in small caps is acceptable; sentence case is the safer default.

### V3.3 — Grid discipline

- Base unit: 4 px at 1× scale. Primary spacing increments: 8, 16, 24.
- **Align knob centers**, not bounding boxes. Optical centering > geometric centering for circular elements.
- Group internal padding ≥ group external padding ÷ 2. (If groups are 24 px apart, knobs inside a group are at most 12 px apart.)
- Establish the grid once. Don't drift mid-design.

### V3.4 — Color in dark UIs

Dark UIs are the default for plugins (studio environments, contrast against DAW chrome, glare reduction). Rules:

- One dominant accent color. Two maximum. More than two and you've lost the hierarchy.
- **Neutrals are not gray.** Use a warm or cool tinted neutral; pure neutral gray reads as unfinished. Pick a temperature and stick with it.
- Reserve saturated color for state and modulation. Static decoration in saturated color is noise.
- Highlights / "white" should be an off-white (e.g. `#F4F1EA` bone). Pure `#FFFFFF` is too hot against most chassis darks and tints all images that share its channel.

### V3.5 — Contrast: WCAG floors apply

- Text against background: 4.5:1 minimum, 7:1 for small or thin type.
- UI elements (knob bodies, control outlines): 3:1 minimum against immediate background.
- Disabled states: explicitly reduce contrast — don't merely desaturate.
- **Dark UIs need more contrast than light, not less.** The eye's adaptation range narrows in dim viewing; cheaping out on contrast in a dark plugin is the most common readability failure.

### V3.6 — Information density (Tufte)

Maximize data-ink ratio for displays (meters, spectrum, envelope curves):

- Remove decorative ticks, gradients, drop shadows that don't carry data.
- Grid lines should be lower contrast than data.
- Peak / important values get higher contrast.
- "Chartjunk" — purely decorative graphical elements on a data display — is forbidden.

For static chrome (chassis, frames, labels) the data-ink rule does *not* apply; chrome can be expressive.

---

## 4. Audio-domain conventions (Jakob's Law applied)

These conventions are non-negotiable defaults. Break only with explicit user approval and visible documentation in the UI.

### C4.1 — Knob interaction grammar

| Input | Behavior |
|---|---|
| Drag (vertical, or radial if explicitly chosen) | Set value |
| Shift + drag | Fine adjustment (≥ 5× slower than normal) |
| Double-click **or** Cmd/Ctrl + click | Reset to default |
| Right-click | Open context menu (Set to default, Type value, MIDI Learn, Copy/Paste value) |
| Scroll wheel | Increment by step (or 1/100 of range if no step defined) |
| Shift + scroll | Fine increment |
| Alt + drag (or Cmd + drag) | Modifier action where the plugin defines one (link L/R, snap to grid) — never overload silently |
| Hover | Show tooltip with value + unit after small delay (~250 ms) |
| Drag start | Show floating value readout near cursor or in fixed display |

Provide **both** double-click reset and modifier-click reset. Users come from different DAWs and assume different defaults; offering both costs nothing.

### C4.2 — Value display

- Show the current value with unit during drag (and for ~600 ms after release).
- Format: `-12.3 dB`, `440 Hz`, `1.20 ms`, `+50%`. Use the unit's conventional notation, not internal floats.
- Decimal precision: enough to show the smallest user-perceivable change, no more. `0.1 dB` for gain; whole `Hz` for frequencies > 100 Hz, decimal Hz below; `ms` to 1 decimal up to 10 ms then whole.
- Bipolar values display sign explicitly: `+3.0 dB`, `-3.0 dB`, `0.0 dB`.

### C4.3 — Bipolar parameters

- Visual fill / indicator originates at center (12 o'clock or 6 o'clock by convention; pick one per product).
- A small center detent ("notch") helps the user find zero.
- Default position is center; never default a bipolar parameter to a non-zero value without strong reason.

### C4.4 — Meters and scopes

- Refresh rate: 30 FPS minimum, 60 FPS preferred. Below 30 FPS reads as broken.
- Peak hold: 1–2 s, with a faster falling peak indicator.
- Clip indicators: latch until clicked or for ≥ 2 s.
- Scale: dBFS by convention, with 0 dBFS visually distinct (a hard line, not just a tick). Leave at least 6 dB of headroom visualization above the typical signal level.
- Color zones: green (safe) → yellow (hot) → red (clipping) is the universal pattern. Override only with reason.
- Spectrum displays: log frequency axis, dB amplitude axis. Linear frequency is wrong for music.

### C4.5 — Signal flow

- Left-to-right is "earlier in the chain → later." Top-to-bottom is acceptable for tall layouts.
- Modulation sources visually feed *into* targets (arrows, lines, halos).
- Sidechains and feedback paths should be visually distinct from primary signal flow.

### C4.6 — Modulation visualization

When a parameter is being modulated:

- Show the *modulation range*, not only the current modulated value (a halo, secondary ring, or shaded arc).
- The base value indicator remains visible; the modulated value moves within the halo.
- Multiple modulation sources sum visually if possible; if not, indicate "modulated by N sources" with a count.

### C4.7 — Presets and A/B compare

- A/B compare is a single click with visible state (which slot is active).
- Preset name visible at all times when a preset is loaded; modified state shown ( `*` suffix or color change).
- Save / Save As / Init Patch reachable in ≤ 2 clicks.

### C4.8 — Resize and DPI

- Integer scales preferred (1×, 1.5×, 2×). Fractional scales produce blurry baked assets.
- Persist the user's choice in host state.
- Resize must not break layout — define a single source-of-truth size and scale it; never re-flow at different scales.
- Some hosts manage scale globally (modern DAW + HiDPI); respect host scale where the framework signals it.

### C4.9 — Keyboard and MIDI

- Tab order, if implemented, follows reading order (left-to-right, top-to-bottom).
- Numeric value entry on right-click or double-click is expected for any continuous parameter.
- MIDI Learn (right-click → "Learn"), where supported, must show a visible "learning" state and confirm the binding.

---

## 5. House style — Hyperfocus DSP / REXIST

This is the visual stance for *this developer's* plugins. Other plugin makers will have other stances; the principles in §§1–4 apply universally.

### H5.1 — Hardware-referential metaphor

Plugins look like physical instruments: dark chassis, knobs with caps and skirts, hex screws at corners, segment-style numeric displays, indicator LEDs. **Restraint required:**

- No faux wood grain, faux leather, faux brushed metal as primary surface. A subtle texture is OK; literal photo-skeuomorphism is not.
- Screws are accents, not features. Don't draw attention to them.
- The metaphor should make the plugin *feel* like an instrument; it should not become a museum diorama of one.

### H5.2 — Palette

- **Chassis dark:** consistent within a plugin, ~`#1A1A1E` to `#2A2A2E` range.
- **Bone white:** `#F4F1EA`. This is the highlight / text / brand color. **Not `#FFFFFF`.** Pure white reads cold and tints image assets darker when used as a multiplier.
- **Accent:** one warm or cool saturated color per plugin (the plugin's "voice"). Use sparingly: indicators, active states, brand mark.
- **Semantic colors:** green / yellow / red strictly for meter zones, clip lights, error states.

When tinting image assets that should appear in their natural baked color, use **identity white** (`#FFFFFF` as the tint multiplier), not the brand bone. Brand bone is for *text and chrome*, not for tinting prebaked artwork.

### H5.3 — Typography

- **Numerics:** 7-segment LED rendering for primary values. Provide letter fallbacks for the full alphabet (`M → n`, `W → u`, `K → h`, etc.) so display strings degrade gracefully.
- **Labels:** sans-serif, 10 pt baseline at 1×. Consistent across the plugin.
- **Brand wordmark:** custom or selected display face — used only for the brand mark, never body labels.

### H5.4 — Knobs

- Photoreal **baked** (e.g. Cycles render exported as filmstrip or per-angle frames). Not painted in real time.
- Knob angle range conventional: ~270° (5 o'clock to 7 o'clock through top).
- Tick dots scale with knob size; very small knobs may drop tick dots entirely.
- Indicator line / dot on knob cap is the primary value cue; the body texture is decoration.

### H5.5 — Brand mark

- Plugin name + version badge live at the top of the chassis.
- Badge placement: left of the wordmark.
- Use the established brand gag where applicable (e.g. inverted-N in NINER's "9" badge). Don't reinvent per plugin.

### H5.6 — Coherence over novelty

**Pick a stance per plugin, then commit.** Within one plugin do not mix:

- Skeuomorphic chassis with flat-design controls.
- Photoreal knobs with vector switches.
- LED segments with system-font numerics.
- Two different knob caps.

Cross-plugin coherence is a secondary goal; intra-plugin coherence is non-negotiable.

---

## 6. Process discipline

### Pr6.1 — One feature, one scope

Implement exactly the change requested. Do **not** refactor adjacent code, polish unrelated controls, rename variables you didn't touch, or "fix" things the user didn't ask about. If you see a problem outside scope, raise it; don't fix it silently.

### Pr6.2 — Layout lock

Once the user has approved a layout (positions of knobs, panels, sections), **do not move them** without explicit re-approval. Visual muscle memory is real; moving a knob a user has learned is a tax. Small alignment fixes within a pixel or two are OK; re-flowing rows is not.

### Pr6.3 — Iteration discipline

Step up fidelity progressively:

1. **Spec/ASCII** — describe layout in text, agree the shape.
2. **Low-fi** — blocks and labels, no final styling.
3. **High-fi** — final styling, placeholder assets where final ones aren't ready.
4. **Final assets** — baked renders, real textures.

Do not jump to step 4 on first attempt. Re-work at step 4 is expensive; re-work at step 1 is cheap.

### Pr6.4 — Visual verification before "done"

**Type-checking is not visual verification.** Tests passing is not visual verification. Before claiming UI work is complete:

1. Build / load the plugin in a host (or standalone).
2. Open it. Look at it.
3. Interact with every changed control.
4. Resize, switch presets, A/B compare.
5. Compare to the previous version (screenshot) if possible.

If you cannot run a build (sandboxed, no display, missing dependency), **say so explicitly** — do not claim success without verification.

### Pr6.5 — Naming hygiene

- Component / class / id names must not collide with framework reserved names or with names used by mockup / preview tooling. (Example: a CSS class named `.header` may collide with a preview frame's own header style; pick `.bar-top` or `.title-strip` instead.)
- Choose names that describe role, not appearance (`mod-source-row`, not `green-bar`).
- Asset paths consistent across plugin: `assets/knobs/large/`, `assets/chassis/`, etc.

### Pr6.6 — No drive-by aesthetic preferences

If you have an opinion about how the chassis should look that differs from the established direction, **raise it** as a question. Do not "improve" the look as part of an unrelated edit. Aesthetic changes are content; content changes need consent.

### Pr6.7 — Naming things the user asked you not to name

Brand names, plugin names, parameter names, and version numbers are decided by the user. Do not invent them, do not "correct" them, do not shorten them, do not silently rename. The same applies to colors and palette tokens.

---

## 7. Hard rules (NEVER / ALWAYS)

These are the recurring footguns. Treat as imperatives.

### Never

1. **Never move a user-approved knob row** without explicit request. (See Pr6.2.)
2. **Never auto-shrink text to fit.** Fix the layout. (See V3.2.)
3. **Never default a bipolar parameter** to a non-zero value without strong reason. (See C4.3.)
4. **Never use pure `#FFFFFF`** as the brand highlight color. Use bone (`#F4F1EA`) for chrome/text; use identity `#FFFFFF` only as an image tint multiplier. (See H5.2.)
5. **Never mix skeuomorphic and flat** controls within the same plugin. (See H5.6.)
6. **Never claim UI work is done** without running and visually verifying. (See Pr6.4.)
7. **Never invent or alter** plugin names, parameter names, brand marks, or version numbers. (See Pr6.7.)
8. **Never name a component class** with a name that may collide with framework or tooling reserved names (`header`, `nav`, `main`, etc. in CSS-influenced contexts). (See Pr6.5.)
9. **Never block the audio thread** for UI state. Cache parameters on the GUI side; read latest atomic snapshot.
10. **Never ship saturated decorative color.** Saturated color is state. Decoration is neutral. (See V3.4.)
11. **Never use linear frequency axes** for spectrum displays. Log. (See C4.4.)
12. **Never overload a modifier silently.** If Alt+drag does something special, that must be discoverable from tooltip or docs. (See C4.1.)
13. **Never auto-resize layout** when DPI changes. Scale, don't re-flow. (See C4.8.)
14. **Never fabricate specifics** in user-facing copy (release dates, durations, anecdotes). Confirm with the user.

### Always

1. **Always cite the rule or principle** (`P1.3`, `R7.4`) when justifying a UI decision in commits or PR descriptions.
2. **Always render values with units.** No bare numbers. (See C4.2.)
3. **Always provide both double-click reset and modifier-click reset.** (See C4.1.)
4. **Always show a tooltip with current value** on hover for every continuous parameter. (See C4.1, P1.6.)
5. **Always meter to at least 30 FPS** with peak hold and clip latch. (See C4.4.)
6. **Always integer-scale** baked assets across DPI. (See C4.8.)
7. **Always group with proximity first**, boxes only as a fallback. (See V3.1.)
8. **Always close both gulfs** (execution & evaluation) for every parameter. (See P1.8.)
9. **Always raise out-of-scope problems** to the user. Don't fix silently. (See Pr6.1.)
10. **Always step up fidelity progressively.** Don't jump to final renders on first attempt. (See Pr6.3.)
11. **Always surface conflicts** between this document and the current task before acting. (See §0.)

---

## 8. Pre-completion checklist

Run this checklist before declaring any UI work complete. Answer each in writing (in your response to the user) or skip with explicit reason.

### Function

- [ ] Every primary control has a visible label.
- [ ] Every primary control has a value readout with unit.
- [ ] Every continuous parameter responds to: drag, Shift+drag, double-click, right-click, scroll, Shift+scroll.
- [ ] Double-click and Cmd/Ctrl+click both reset to default.
- [ ] Bipolar parameters fill from center; default is zero.
- [ ] Disabled controls are visibly disabled (reduced contrast, not just desaturated).
- [ ] Tooltips appear on hover for every continuous parameter.

### Feedback

- [ ] Knob drags produce visible response within one frame.
- [ ] Meters refresh ≥ 30 FPS; peak hold present; clip latches.
- [ ] Modulation, where present, is visualized as a range or halo, not just a moved value.
- [ ] A/B compare state is unambiguous at a glance.
- [ ] Preset modified state is shown.

### Composition

- [ ] One dominant accent color; no decorative saturation.
- [ ] Highlights / text use bone white (`#F4F1EA`), not pure white. (House style.)
- [ ] Knob centers aligned (not bounding boxes).
- [ ] Proximity grouping before bounding boxes.
- [ ] Numerics use tabular figures or monospaced; no horizontal jitter on change.
- [ ] Contrast meets WCAG floors (4.5:1 text, 3:1 UI).

### Convention

- [ ] Signal flow reads left-to-right (or top-to-bottom for tall layouts).
- [ ] Spectrum axes are log-frequency, dB-amplitude.
- [ ] Knob CW = increase. Slider up = increase. No reversed mappings.
- [ ] No modifier overloads without tooltip or doc.

### Identity (house style)

- [ ] Skeuomorphism level consistent across every element (no mixed flat + skeuo).
- [ ] Knobs are baked photoreal, not real-time-painted.
- [ ] Brand mark placed correctly with the established badge gag.
- [ ] Palette within the chassis-dark / bone / single-accent system.

### Process

- [ ] Scope matches the user's request. No drive-by edits.
- [ ] No moved knob rows or re-flowed sections without explicit re-approval.
- [ ] Names do not collide with framework / tooling reserved names.
- [ ] Build runs and the plugin loads in a host or standalone.
- [ ] You have opened the plugin and visually inspected the change. (Or stated explicitly that you could not.)

### Communication

- [ ] Justifications cite the rule (`P1.3`, `R7.4`, etc.).
- [ ] Any conflict between this doc and the task was surfaced before action.
- [ ] No fabricated specifics (dates, durations, anecdotes).

---

## Appendix A — Vocabulary

| Term | Meaning |
|---|---|
| **Chassis** | The main body/background of the plugin. Hardware metaphor. |
| **Knob** | Rotary continuous control. Cap (top) + skirt (side) + tick(s). |
| **Fader / slider** | Linear continuous control. |
| **Encoder** | Rotary control with no fixed endpoints (relative). Rare in plugin UI; common in MIDI hardware. |
| **Segment / 7-seg** | Numeric (and limited alpha) display rendered as illuminated bars. |
| **LED** | Single-color state indicator, on/off or intensity. |
| **Badge** | The brand mark / version indicator, typically corner of chassis. |
| **Wordmark** | The plugin name set in display type. |
| **Halo / ring** | Secondary ring around a knob showing modulation range or value. |
| **Detent** | A felt-or-visual snap point at a particular value (often center). |
| **Filmstrip** | Pre-rendered knob frames stacked as a single image, indexed by value. |
| **Bake** | A pre-rendered asset (vs. real-time drawn). Used for photoreal knobs, chassis textures. |
| **Sidechain** | Secondary signal input to a processor. Visualized distinctly from primary path. |
| **A/B compare** | Two-slot snapshot of all parameters with one-click toggle between them. |

---

## Document changelog

- v1.0 — Initial version. Principles, laws, conventions, house style, process, rules, checklist.
