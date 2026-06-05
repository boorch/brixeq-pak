# BRIXEQ

A handheld groovebox and step sequencer for the TrimUI Brick. Five tracks, sixteen patterns per track, sixteen scenes per project, with per-step snapshots of the whole instrument so every step can sound completely different from the next.

---

## 1. What is BRIXEQ

BRIXEQ is a tracker-ish step sequencer with **per-step instrument snapshots**: every step on every track carries its own engine, envelope, filter, FM parameters, noise type, chord stack, send levels, probability, and trigger logic. There is no "patch" or "instrument slot" to switch between, each of the 16 steps in a pattern is its own complete sound.

The trade-off this buys you: you can build hugely expressive 16-step patterns that morph through different timbres without managing instrument slots, and you can copy any single step onto another to clone its entire instrument character.

Five concurrent tracks let you stack a beat, a bass, a lead, a pad, and a percussive layer all at once. Master-bus delay, reverb, and compressor handle the glue.

This whole **per-step instrument snapshots** idea (and many others you'll find throughout BRIXEQ) is basically based on FMS, a suepr fun FM groovebox for GBA, created by Fors. Check it out here: https://lo-bit.club/fms

BRIXEQ is specifically developed for TrimUI Brick. You may run it on other Linux systems on various handheld devices, but your mileage may vary, and it's unsupported.

---

## 2. The data model at a glance

A project is built out of four nested concepts. The most important thing to internalise: **patterns are the only place per-step instrument data lives**. Scenes don't own patterns, they just point to which pattern from each track's bank is currently active. Songs don't own patterns either, they reference scenes.

### The big picture

```
PROJECT
│
├── TRACKS  (five concurrent voices: T1 T2 T3 T4 T5)
│     │
│     └── each track owns a BANK OF 16 PATTERNS:
│
│            ┌──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┐
│            │ 0│ 1│ 2│ 3│ 4│ 5│ 6│ 7│ 8│ 9│ A│ B│ C│ D│ E│ F│
│            └──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┴──┘
│
│            each pattern is 16 STEPS + pattern settings
│            (length, direction, clock div, default engine, filter mode)
│
│            each step is a full instrument SNAPSHOT
│            (note, engine, chord, envelopes, filter, FM/noise params,
│             level, pan, sends, probability, trigger logic)
│
├── SCENES  (sixteen of them: 0 through F)
│     │
│     └── each scene stores ONLY:
│            • which pattern is active per track  (5 numbers 0..F)
│            • scale  (root + mask)
│            • a transpose strip per track  (5 strips)
│            • master FX state  (delay, reverb, compressor)
│
│            scenes do NOT store the pattern data itself, just the
│            pointers. swapping scenes = "play a different combination
│            of the patterns I already have, with a different scale /
│            transpose / FX recall".
│
└── SONG  (up to 64 rows)
      │
      └── each row references a scene (or none) + a duration in bars.
          fires its scene at the start of its window.
```

### Why this matters

- **Pattern data is shared.** Editing T1's pattern 3 affects every scene that has T1 pointing to pattern 3. So if two scenes share the same drum loop on T1, you only author it once. If you wanted them to differ, you'd point one scene at T1 pattern 3 and another at T1 pattern 4, then author each.
- **Scale, transpose, and master FX are per-scene.** Editing scene 0's transpose only affects scene 0; scene 1's transpose is untouched.
- **Scenes are essentially viewing angles on the same set of tracks.** A verse and chorus can use the exact same patterns but with different transposes and FX. Or they can use entirely different patterns. Both work.

### What a scene stores, visually

```
SCENE 2
┌──────────────────────────────────────────────────┐
│  pattern_idx:   T1=4  T2=7  T3=1  T4=0  T5=3     │   ← five pointers
│                                                  │
│  scale:         root=G,  mask=mixolydian         │
│                                                  │
│  transpose:     T1 → [0, +5, +7, +12]            │   ← per-track,
│                 T2 → [0]                         │     each up to 16
│                 T3 → [0, -3]                     │     semitone offsets
│                 T4 → [0]                         │
│                 T5 → [0]                         │
│                                                  │
│  master FX:     delay {time, fbk, tone, mod}     │
│                 reverb {size, decay, diff...}    │
│                 compressor {amount, sc src...}   │
└──────────────────────────────────────────────────┘
```

When you queue this scene and it fires at the next bar: every track instantly starts playing the pattern its pointer references, the scale and transpose strips kick in, and the master FX swap. The master clock is preserved, only the source patterns and the FX state change.

---

## 3. First sound

1. Launch BRIXEQ. The splash screen shows for about 3 seconds; any button skips it.
2. The last-saved project loads automatically. A fresh install starts with an empty project (every pattern silent).
3. Press **START** to begin transport. Nothing audible yet, no steps are placed.
4. Move the cursor onto any step cell with the d-pad.
5. Hold **B** and tap **↓** to place a note at that step. The step lights up; transport will trigger it on the next pass.
6. Repeat across other steps and tracks.

The transport keeps running across edits. There is no "stop to edit, play to listen" mode switch.

---

## 4. The Main Editor

The default screen. Top to bottom:

- **Topbar**: project name (left), scale quantizer, BPM, shuffle, modulator-page icon, master-page icon (right). The scale, BPM, and shuffle cells are editable; the mod icon opens the modulator screen, the master icon opens the master FX page.
- **Track row**: for each of the five tracks, a track header (mute, spectrum), pattern slots (the 16-pattern bank as a vertical strip), the **step grid** (the 16-step pattern), and the **track footer** (length, direction, clock-div for the focused pattern).
- **Left and right decks**: when a step is focused, both sides slide on screen to show parameter cells. The **left deck** carries engine-common params (pitch, logic, envelope, filter, output and sends). The **right deck** is engine-specific (different controls for VA vs FM vs the drum engines).
- **Scenes strip**: sixteen scene cells along the bottom, just above the footer.
- **Footer**: transient toasts, mode label, hints.

Pressing **X** on the step grid cycles between three column views: **steps → patterns → transpose**. Steps view is the default; the others let you edit the pattern bank or the per-track transpose sequence inline without leaving the main editor.

---

## 5. Cursor and navigation

- D-pad **moves the cursor**. The cursor is one-cell, it lives on a specific track and column.
- **L1 / R1** focus the left or right deck respectively. The cursor jumps into the deck's currently-active row; releasing returns it to the step grid.
- **Topbar** is reached by pressing **↑** while on the top step of any track; the cursor climbs out of the grid into the scale, BPM, shuffle, mod, and master cells (five stops, ←/→ to move between them).
- **Scenes strip** is reached by pressing **↓** below the last step.
- **X** cycles the column view (steps → patterns → transpose → back).

The cursor never wraps around the edges. It clamps at the boundaries of the active region.

### Transport

- **Bare START** plays from the **cursor's step position**. If the cursor is on step 8, every track's pattern starts at its step 8 (clamped to each pattern's length). Lets you audition a section without rewinding to the beginning.
- **SELECT + START** forces playback to begin at **step 0** of every pattern, regardless of where the cursor sits.
- **START while playing** stops the transport.
- If the cursor isn't on a step (you're on the topbar or scenes strip), bare START falls back to step 0.

---

## 6. Editing steps

The fundamental editing chord is **B held plus d-pad**, always with the cursor on a **step**. The cursor stays put; the value of the *currently-active parameter* on that step changes.

When you first land in the editor, the active parameter is **note**, so the gesture below means "edit pitch". To edit a different parameter (level, filter freq, harmonics, etc.), hold **L1** or **R1** to enter the deck and pick the parameter cell (see §7). When you release L1/R1, the cursor returns to the step grid but the parameter you picked stays selected. B + d-pad on a step now edits that parameter's value at that step.

- **B + ↓** on an empty step: place a note at the focused cell, using the previously-edited pitch (or middle C the first time).
- **B + ↑/↓** on a placed step: coarse change (e.g. ±12 semitones on pitch).
- **B + ←/→** on a placed step: fine change (e.g. ±1 semitone on pitch).

Vertical typically equals coarse, horizontal equals fine, on parameters where that distinction makes sense. On single-axis parameters, both axes step by 1.

### Auditioning a step

A single **B-tap** (no arrows) on the focused step auditions it: the step's full snapshot (engine, pitch, chord stack, envelopes, filter, every per-step param) plays once immediately. Bypasses probability, logic conditions, trigless, and track-muted filtering, so you always hear what's authored at that cell.

This is the fastest way to decide what to cut, copy, or paste. The whole BRIXEQ idea is that "a step is a full snapshot," so reaching that snapshot by ear should be one tap away.

B-tap layers cleanly with the existing place / trigless behaviors:

- **B-tap on an empty step**: places a note at the previously-edited pitch, then auditions the just-placed note.
- **B-tap on a populated step**: auditions it.
- **B-tap twice on the SAME populated step** within a short window: toggles trigless, then auditions (preview still plays even on trigless cells, since the audition path ignores the trigless flag).

A trigless step is **a parameter lock**, not just a placeholder. When the playhead lands on a trigless cell, the step's full snapshot (filter freq/Q/env depth, FM or VA or drum body params, pan, level, delay/reverb sends) gets pushed onto the currently-sounding voice on that track without re-attacking any envelope. So you can author smooth filter sweeps, FM index morphs, drum body tilts, or volume / pan moves over the held note. The amp / FM / filter envelopes keep their existing state; only the continuously-read fields change.

Pitch is intentionally left alone (since the note isn't re-triggered) and modulators with TRIG / ONCE run modes are not re-armed by a trigless step. Use trigless when you want the param changes; use a real trig when you want to retrigger.

### Editing topbar cells

You can edit the **scale**, **BPM**, and **shuffle** cells in the topbar the same way as a step: navigate the cursor up into the topbar, focus the cell, then B + d-pad.

### Shuffle

A project-wide swing percentage in the topbar (reads `≈50%` next to the BPM). Range 50% (straight, no swing) to 65% (heavy MPC-style swing). Every odd-indexed step (the 2nd, 4th, 6th, ... in 1-indexed counting, the "off-beats" of each pair) has its grid window delayed; even-indexed steps stay on grid. At 50% the pattern plays straight; at 65% the off-beats land roughly 30% later than nominal.

- **B + ←/→** edits by ±1%.
- **B + ↑/↓** edits by ±5%.
- **A** on the shuffle cell resets to 50% (straight).

Shuffle moves the **grid**. Microtiming (per-step nudge, see §7) is applied on top of the shuffled grid: a swung off-beat with a `+33%` microtime ends up even later, not relative to the unshuffled position.

### Selection

Hold **Y** while moving the d-pad to drag a selection rectangle from the cursor. The cursor follows the far corner of the selection. Releasing Y leaves the selection in place; pressing Y again extends it from where you left off.

Multi-tap behavior on Y:
- One tap (no movement): clear selection.
- Two taps quickly: select the whole track at the cursor column.
- Three taps: select the entire grid.

Pressing any bare d-pad arrow clears the selection and resumes single-cell cursor navigation.

### Copy / cut / paste / undo / redo

A single button, **A**, does both, deciding by context:

- **A on a populated step**: **cut**. The step is captured into the clipboard and cleared.
- **A on an empty step with clipboard primed**: **paste**.
- **A with a multi-cell selection that contains anything populated**: multi-cut.
- **A with a multi-cell selection that is all empty plus clipboard primed**: multi-paste at the selection's top-left.

Double-tap A on the same populated step equals cut plus paste-back, which equals **copy** (the source is restored, the clipboard stays primed for stamping elsewhere). The same idiom works on multi-cell selections.

For direct copy / paste without the cut detour, use the **L2 / R2** triggers:

- **L2**: COPY the focused step (or selection) into the clipboard. Source untouched. Empty cells still copy, the resulting empty clipboard turns R2 into a "wipe step" stamp.
- **R2**: OVERRIDE PASTE. Writes the clipboard at the cursor regardless of whether the target is populated. Empty cells in a multi-cell clipboard CLEAR their target. Cursor advances past the pasted region just like A-paste does.

The clipboard captures the **whole step snapshot** (engine, every param, chord, logic, the lot), so a paste recreates the source step's instrument character exactly.

### Randomize selection

With a multi-cell step-grid selection alive, **L1 + R1 + face button** randomizes the selected cells in place. Four escalating levels mapped to the four face buttons:

- **L1 + R1 + B** (south): level 1, density stamp from clipboard only. No mutation.
- **L1 + R1 + Y** (west): level 2, mutate the currently-focused step parameter on populated steps in place. Empty cells stay empty.
- **L1 + R1 + X** (north): level 3, density + pitch (scale-quantized) + engine (subtle right-deck variation).
- **L1 + R1 + A** (east): level 4, density + pitch + engine + full per-engine left-deck and right-deck randomization.

While both shoulders are held with a valid selection (≥ 2 cells on the step grid), the visor returns to the selection and the footer reads `RANDOMIZE?`. Pressing a face button in this "armed" state fires the randomize at that level instead of the button's normal action (paste / copy / view-cycle / selection-start). Releasing either shoulder reverts to the normal single-deck focus.

**Base step**. The clipboard provides the seed: if you've L2-copied a single step, that step's full snapshot (engine, every param, chord, all of it) is what level 1 stamps and what level 3 / 4 mutate from, and is also the anchor that level 2 perturbs around. With an empty clipboard, the seed is the default step (VA, C4, factory defaults).

**Levels 1, 3, 4 wipe-and-fill**. Each press of L1 / L3 / L4 first wipes every cell in the selection, then refills according to the density cycle. **Level 2 is in-place**: it touches only steps that are already active (or trigless), leaves empty cells alone, and never creates new steps. Level 2's "what gets randomized" is determined by the currently-focused parameter on the decks (the same icon B + d-pad would edit): pitch, engine, filter cutoff, FM amount, drum timbre, chord, etc.

**Density cycling** (levels 1, 3, 4). Consecutive presses of the SAME level button against the SAME selection walk the fill count, wrapping 1 → 2 → 3 → 4 → 1:

- Press 1: 25 % of the selection length.
- Press 2: 50 %.
- Press 3: 75 %.
- Press 4: 100 %.
- Press 5: back to 25 %, etc.

**Level 2 range cycling**. Consecutive same-(level, active param) presses widen the ±N window around the clipboard's value at that field, wrapping 1 → 2 → 3 → 4 → 1. The window is calibrated for a full 0-FF byte (cycle 1 = ±5, cycle 2 = ±10, cycle 3 = ±30, cycle 4 = ±60) and **rescaled to the field's own range** for narrower params, so a small field like `probability` (0-3) or `noise mode` (0-4) still gets a real ladder instead of saturating on every cycle (cycle 4 reaches roughly the full span; lower cycles ladder up to it). The result is clamped to the field's legal range. Special cases:

- **`note`**: the offset is in semitones, scale-snapped on melodic engines (VA / FM), raw on drums.
- **`microtime`**: walks the 7 legal detents (▲50 … ▼50) so it never lands on an unreachable in-between value.
- **`engine`**: not a ±N walk (a 6-value list would bias toward the ends). Instead the cycle sets the *chance* the engine changes (25 / 50 / 75 / 100 %); when it changes, the new engine is drawn from the clipboard's own pool (melodic stays melodic, drums stay drums), so a track never scatters across families.
- **`chord`**: see below.

**Level 2 on `chord`**. When the focused param is `chord`, level 2 always rolls a fresh chord for each populated step, regardless of whether the clipboard step actually carried a chord. Here the "detents" are the chord shapes themselves, and the pool widens with the cycle: cycle 1 picks maj / min (50/50); cycle 2 adds 7ths at lower weight; cycle 3 makes the four core triads/7ths equal; cycle 4 is uniform across all nine shapes (maj, min, maj7, min7, sus2, sus4, add9, maj-add9, min-add9). maj / min picks retain the 33 % chance to add a +12 octave as a third voice, same as level 4.

Changing the selection (re-anchoring, resizing), pressing a different level button, or — at level 2 — switching the active param, resets the cycle to press 1.

**Per-track behaviour** (levels 1, 3, 4). If the selection spans multiple tracks, each track is randomized independently. Drum-pool selections at L3 / L4 respect drum convention: BD always lands on the selection's first step, SD is weighted toward steps 4 and 12, and only canonical kick / snare positions are populated.

**Engine pools** (levels 3, 4). The clipboard engine picks the pool:

- VA or FM clipboard → mixed VA / FM steps.
- NOIZ or CY clipboard → mostly CY with occasional NOIZ (~ 8 %).
- BD or SD clipboard → mixed BD / SD honouring drum positional rules.

**Microtime clash protection** (level 4). Adjacent ± 50 % microtime values never collide. If two consecutive filled steps would land at the same moment (a +50 % followed by a -50 %, or vice versa), the second is auto-demoted to 0. Also checked against the steps immediately outside the selection on the same track.

**Undo**. Each press is one undo entry. L1 + L2 reverses the whole operation in one shot, regardless of how many cells changed.

### Undo / redo

- **L1 + L2**: undo the last edit. Walks the snapshot ring back one step. Includes step edits, cut / paste, A-reset, pattern / scene copy, modulator tweaks, BPM, scale, and shuffle changes.
- **R1 + R2**: redo. Walks the ring forward.

A snapshot is captured before each mutating action. The ring holds up to 50 entries. Loading a project clears the ring (the old history refers to a file that's no longer in scope). Hold the shoulder FIRST then tap the trigger to fire the chord; pressing the trigger alone falls through to copy / paste.

---

## 7. Step parameters

When a step is focused, the **left** and **right** decks slide on screen. The decks are **param selectors**, not direct editors. Their job is to pick *which* per-step parameter the B + d-pad gesture is going to edit.

To use a deck:

1. Hold **L1** (left deck) or **R1** (right deck). The cursor enters the deck.
2. With the modifier still held, use the d-pad to move between cells. Up / down picks a row (group); on the left deck, left / right picks between sibling params inside that row.
3. Release the modifier. The cursor jumps back to the step grid. The parameter you focused is now the **active** one.
4. B + d-pad on any step edits that parameter's value at that step.

The active parameter persists across step navigation, so you can scrub left / right across the grid editing the same parameter on every step you land on.

### Left deck (engine-common)

Rows top to bottom, each containing one or more sibling parameters:

- **PITCH**: note, chord stack, slide, pitch-sweep range and duration.
- **RULE**: logic (trigger conditions, e.g. every nth bar, alternating), probability (1/4, 1/2, 3/4, 1/1), and microtime (per-step ±50% nudge off the grid).
- **AMP ENV**: attack, hold, decay (an AHD envelope, no separate sustain).
- **FILTER**: bipolar DJ-style sweep (HP below center, LP above), Q, plus filter envelope attack, decay, depth.
- **OUT**: level, pan, send-delay, send-reverb (per-step amounts into the master bus FX).

### Right deck (engine-specific)

The right deck swaps wholesale when the step's engine changes. Each row holds exactly one parameter, so navigation here is up / down only.

- **VA**: harmonics, timbre, morph, overdrive.
- **FM**: modulator ratio, mod amount, mod feedback, mod attack, mod decay.
- **NOIZ**: noise mode (narrow, pink-ish, white, pink, brown).
- **BD / SD / CY**: drum-engine knobs (tuning, snap, decay, color; the exact set varies per drum).

### Chord stack

A step can play up to **five voices simultaneously**: one root note (the step's pitch) plus four semitone offsets. Offset 0 equals an unused slot, so the count is effectively variable. Voices share the step's engine and envelope; transpose and scale-quantize apply to the whole stack.

This is the chord model: dial in `+0 +4 +7 +12` and the step plays a major triad with octave. Polyphony is **intra-track only**, a fresh trigger on the same track cuts off the previous voices on that track.

### Microtime

Per-step nudge of the audible fire moment off the grid, edited from the **microtime** cell in the RULE row of the left deck. Seven discrete values render as 4-char labels on the cell:

`▲50%` — `▲33%` — `▲16%` — `----` (on grid) — `▼16%` — `▼33%` — `▼50%`

The ▲ glyph means **earlier** in time (above the playhead in the tracker's downward flow); ▼ means **later**. The size is in percent of nominal step duration, so it tracks both BPM and the pattern's clock divider automatically.

- **B + ↑/←** moves toward ▲ (earlier).
- **B + ↓/→** moves toward ▼ (later).
- **A** on a step with microtime resets the cell to `----`.

The visual playhead **always tracks the nominal grid** position regardless of microtime; only the audible event shifts. So a `▲50%` step still highlights at its grid position; the sound just arrives half a step early. Useful for groove humanization, hi-hat tightening, off-beat ghost notes.

Microtime composes with **shuffle** (see §6): shuffle moves the grid (delays every odd-indexed step's grid for swing), then microtime moves the fire relative to that shuffled grid. They stack additively.

Adjacent `±50%` steps (e.g. a `+50%` step right before a `-50%` step) fire at the same moment; both audibly trigger (the engine is intra-track polyphonic, so chord steps and overlapping voices are fine).

---

## 8. Engines

Six engines, selectable per-step:

- **VA**: virtual analog. Three timbre macros (harmonics, timbre, morph) blend continuously between several oscillator characters. Plus overdrive. Pitched.
- **FM**: two-operator FM. Modulator ratio, depth, feedback, attack, decay. Carrier is a fixed sine. Pitched, percussive bite.
- **NOIZ**: noise generator. Several noise types (narrow, white, pink, brown) cover the range from tonal hiss to dark rumble. The lowpass and highpass on the left deck shape it. Mostly unpitched (the note value tunes the narrow noise modes).
- **BD**: 808/909-style bass drum.
- **SD**: 808/909-style snare drum.
- **CY**: hi-hat / cymbal. Covers both sounds via the timbre and morph controls.

Drum engines **ignore the per-scene transpose strip and the scale quantizer**. The drum's note value still tunes the synthesized body (so the step's pitch picks the kick / snare / hat pitch), but the scene-level transpose and scale snap pass would only push those tunings around against the user's intent. The four body params (timbre, morph, harmonics, model) carry the per-step character.

---

## 9. Patterns

Each track has a **bank of 16 patterns**. The scene picks which pattern is active per track. The pattern strip on the main editor visualises the bank as a vertical column to the left of the step grid.

Pattern-level settings (per pattern, not per step):

- **Length**: 1..16 active steps. Shorter patterns wrap sooner; useful for poly-rhythmic interplay.
- **Direction**: forward, reverse, pingpong, pongping, pingpong+ (with rest on the bounce), pongping+, random.
- **Clock divider**: 1..8. Divides the master clock for this track. Clock 2 is half-time, clock 4 is quarter-time, and so on.
- **Default engine**: what new steps placed in this pattern start as.
- **Filter mode**: bipolar LP/HP, LP-only, HP-only, or off (sticky per pattern).

### Pattern launch

In **patterns view** (X to cycle), the column shows the pattern bank. **B-tap** on any pattern cell queues that pattern as the track's next active pattern; it swaps in at the next bar boundary, leaving the master clock undisturbed.

Pattern launches are **per-track and live**: they don't fire a scene, don't change scale or master FX, and don't affect other tracks. Use them for fills, breakdowns, A/B switches.

### Pattern copy / paste

In patterns view, the cursor focuses one pattern cell at a time. **Hold R1 and press A** to copy that pattern; **hold R1 and press B** with the cursor on a different cell (any track, any pattern index) to paste. Cross-track is fine: you can clone a melodic figure from T1 P0 into T3 P5 if you want.

The clone covers everything the pattern owns: all 16 steps with their full instrument snapshots, the pattern length, direction, clock divider, default engine, filter mode. A small toast on the footer confirms each operation.

---

## 10. Scale and Transpose

Per-scene, two layers that quantize pitches before they hit the engine:

### Scale

A root note plus a scale mask (which of the 12 semitones are "in"). Every fired pitch, including chord voices and transposed notes, gets **snapped to the nearest in-scale note**. Chromatic scale equals no quantization.

Set via the **scale cell** in the topbar (cursor up from the top step, focus the scale cell, B plus d-pad to edit). Scale is per-scene, so a scene swap also swaps the scale.

### Transpose sequencer (per-scene, per-track)

In **transpose view** (X to cycle), the column becomes a transpose strip. Each row stores a semitone offset; the strip cycles through them as the song plays, transposing every note by the current offset before the scale-snap.

Strip controls (track footer in transpose mode):
- **Length**: 1..16 active cells.
- **Rate**: how many "events" per advance, paired with the adv-mode below.
- **Adv-mode**: *per-pattern-length* (advance once per pattern loop, slow) or *per-step-count* (advance every N steps, fast).

Transpose is per-track, so different tracks can move through different harmonic motions. Drum engines ignore transpose.

---

## 11. Scenes

Sixteen scenes per project. Each scene snapshot stores:

- The **pattern index per track** (which pattern in each track's bank is active).
- The **scale**.
- The **per-track transpose sequencer**.
- The **master bus FX state** (delay, reverb, compressor settings).

### Switching scenes

The scenes strip lives along the bottom. **B-tap** on any scene cell **queues that scene**; it fires at the next bar boundary, swapping all five tracks' patterns plus scale plus transpose plus master FX simultaneously, while keeping the master clock running.

Queued scenes pulse on the strip. Pressing B again on the same cell un-queues. Pressing B on a different cell replaces the queue.

### What scenes are for

- **Song sections**: verse, chorus, bridge as three scenes, with the same patterns but different transposes and FX.
- **Live performance**: pre-bake a setlist of scenes, swap between them mid-show.
- **Variations**: same patterns, different scale (major vs minor) or different master compressor character.

### Scene copy / paste

Same chord as patterns, on the scenes strip: **R1 + A** on the focused scene cell copies that scene into the clipboard, **R1 + B** on a different scene cell pastes. The clone covers everything the scene stores: the five pattern_idx values, scale, transpose strips, master bus settings, and all 8 modulator slots. Spin a variation by copying scene 0 into scene 1 and then editing one parameter.

Scenes are also the conceptual unit that song mode references (see §13).

---

## 12. Master bus

Per-scene. Two parallel send FX plus a master compressor.

### Delay

Ping-pong, tempo-synced. Parameters:
- **Time**: 16 musical divisions (1/32 to 4 bars).
- **Feedback**: how much of the delayed signal feeds back.
- **Tone**: bipolar HP/LP tilt on the feedback path.
- **Mod**: wobble on the delay time (chorus-y when shallow, warpy when deep).
- **→ rvb**: delay output fed into the reverb send. Lets the delay tail bleed into the reverb without explicitly sending the dry signal to both.

### Reverb

Stereo plate-ish reverb. Parameters:
- **Size**: room size, 10..30 m.
- **Decay**: 0.1..32 s tail length.
- **Diffusion**: early reflection density.
- **Mod**: modulation rate (chorused reverb).
- **Tone**: bipolar tilt on the reverb output.

### Master out

- **Gain**: overall master output level.
- **SC src**: sidechain source. Off (uses the internal HP detection) or a specific track 1..5 whose output drives the compressor's detector. Pump-on-kick lives here.
- **SC filt**: bipolar HP/LP tilt on the sidechain detection path.
- **Comp**: single-knob compressor amount. Higher equals more glue.

Each step has its own **send-delay** and **send-reverb** amounts (left deck, OUT group), so the bus FX are triggered per-trigger by the step's amp envelope. Short drum steps send a single tap, sustained pad steps hold the send open.

### Live feedback

The master page shows a live gain-reduction readout and a peak-decimated mono scope of the master output. Press the **master icon** in the topbar to open the page.

### Master copy / paste

On the master page: **R1 + A** or **L2** copies the **active scene's** master FX block. Swap to a different scene (the strip lives at the bottom so you can do it in-place), then **R1 + B** or **R2** pastes onto the now-active scene. The toast in the topbar confirms which scene received the paste. Either gesture is undo-able via L1+L2.

---

## 13. Modulators

Each scene carries 8 **modulator slots**. A modulator is a small LFO (low-frequency oscillator) that nudges a chosen parameter on a chosen track over time: pitch wobble, filter sweep, drum body morph, tempo-synced volume duck, anything a single number on a step can do.

Modulators live alongside transpose, master FX, and pattern_idx in each scene, so swapping scenes swaps the whole modulator setup with everything else. Scene changes (queued live or via song mode) restart every modulator cycle from scratch.

### Entering the mod view

Two entry points, both equivalent:

- **MOD icon** on the topbar (between BPM and MASTER). Hover with the cursor, press B.
- **L3**: toggles into and out of the mod view from anywhere.

The scenes strip lives at the bottom of the mod view too, so you can see at a glance which scene is active.

### Mod copy / paste

On the mod page: **R1 + A** or **L2** copies the **active scene's** 8 mod slots. Swap to a different scene (the strip lives at the bottom so you can do it in-place), then **R1 + B** or **R2** pastes onto the now-active scene. The toast in the topbar confirms which scene received the paste. Both gestures are undo-able via L1+L2.

The view is a vertical list of 8 slots. Each row has a curve preview on the left and a strip of cells on the right.

### Per-slot fields

| Field | Meaning |
|------|---------|
| **TR** | Target track (T1..T5). Any slot can affect any track. |
| **D** | Destination parameter. See the destination list below. |
| **SP** | Speed. One full LFO cycle takes anywhere from 1/16 note to 8 bars. |
| **WV** | Waveform: sine, triangle, saw up, saw down, square, S&H, noise, constant. |
| **DP** | Depth. Bipolar; positive depth adds, negative subtracts. Depth **0** means the slot is off (curve drawn faintly to show this). |
| **RN** | Run mode. FRE (free-running), TRG (resets on each note), ONE (one cycle per note, then freezes). |
| **SM** | Smoothing. Rounds sharp transitions on square/S&H/saw waveforms. |
| **SK** | Skew. Bipolar; warps the waveform left or right so a triangle becomes more saw-like, a sine becomes lop-sided, etc. |
| **PL** | Polarity. BI (centered on zero, swings both ways) or UNI (only above zero, suits envelopes). |
| **VF** | Vertical offset. Shifts the whole curve up or down before depth scaling. |
| **OF** | Phase offset. Sets where in the cycle the slot starts when the scene fires. |

### Destinations

The **D** cycle includes both **universal** params (work on every engine) and **engine-specific** params (only take effect when the target track is playing a step on that engine).

Universal (always live):

- Pitch, Volume, Pan, Filter Freq, Filter Q, Filter Env Depth, Delay Send, Reverb Send.

Engine-specific (silent when the target track isn't using that engine):

- **FM**: mod ratio, mod amount, mod feedback, mod attack, mod decay.
- **VA**: harmonics, timbre, morph, overdrive.
- **BD / SD / CY** (drums): timbre, morph, harmonics, model.
- **NOIZ**: mode (continuous LFO modulation of a 5-way enum is meaningless, so noise mode is author-able but doesn't actually modulate).

Pick any destination freely. If the engine on the target track at fire-time doesn't own that parameter, the slot just doesn't contribute (no error, no noise, no surprise). As soon as a step on that engine fires, the modulation kicks in.

**Pitch modulation respects the active scale.** When a slot targets Pitch, the modulator's bipolar offset is added to the note before the scale's snap pass runs. So on a C major scene, even a +6 semitone modulator nudge lands on a major-scale note (here F or G), not on the F♯ in between. Set the scale to chromatic in the topbar if you want the raw shift through.

### Run modes in detail

- **FRE** (free-running): the LFO ticks continuously from the moment the scene fires. Independent of track triggers. Best for atmospheric drift, slow filter sweeps, breathing pads, evolving room sounds.
- **TRG** (re-trigger): every real trig on the target track snaps the LFO's phase back to its start and lets it cycle from there. Good for per-note articulation: pitch envelopes, filter snappiness on each hit.
- **ONE** (one-shot): like TRG but the cycle freezes at its end value. The LFO is essentially an envelope shape now. Combine with S&H (see below) for per-trigger random offsets.

### Sample & hold

Selecting **S&H** for a slot rolls a fresh table of **16 random values** for that slot. The table travels with the project and stays put: re-rolls only happen when you pick S&H again. Each new roll is guaranteed to differ from the previous one by at least 25% of its values, so you never get a near-identical pattern back.

The roll is **per slot**: each of the 8 slots has its own 16 random values, and they're all guaranteed distinct within a session.

How the 16 values get used depends on the run mode:

- **FRE + S&H** and **TRG + S&H**: the cycle walks the full 16-step pattern across each cycle. The waveform looks like a stair-step.
- **ONE + S&H**: a single value per trigger, advancing one step through the 16 on every retrigger. Great for "random pitch every note" with a guaranteed-bounded set of values.

### Resets and copy

- **A** on a focused cell resets that single field to its default. Track → T1, destination → none, speed → 1/4, waveform → sine, depth → 0, run mode → FRE, polarity → BI, smoothing / skew / phase offset → centred.
- Modulators travel with their scene, so the **scene copy/paste** gesture (R1+A copy, R1+B paste on the scenes strip) duplicates all 8 modulator slots along with everything else.

### Tips

- Subtle modulation usually means **low depth + slow speed + smoothing turned up**. Depth at ±10..±20 is plenty for most "alive" parameter motion.
- A drum track with **ONE + S&H on pitch** at a small depth gives every hit a slightly different tuning, which dodges machine-gun feel.
- A pad with **FRE + sine on Filter Freq**, slow (1 bar+), modest depth, brings a static patch to life.
- Two slots on the same track but at different speeds + different shapes layer into complex motion. Eight slots is a lot.
- **L3** is the fastest way in and out. Start playing, hit L3 to tweak a modulator, L3 again to return.

### Arpeggiators via pitch modulation

Because pitch modulation is scale-snapped, a slot targeting pitch effectively becomes an **arpeggiator**: the LFO sweeps the pitch range, but only in-scale notes come out. The waveform decides the arp shape:

- **Saw up** + depth +12 → ascending arp across an octave.
- **Saw down** + depth +12 → descending arp.
- **Triangle** → up-then-down arp.
- **Square** → octave (or any interval) bounce.
- **S&H** → random arp drawn from the 16-step table.

Speed sets the arp rate (musical division). Depth sets the range. Adjust the scene's scale and the arp shifts with it.

Combine with a TRG run mode so the arp restarts on each note, or FRE so the same arp pattern plays beneath whatever you trigger.

---

## 14. Song mode

A standalone linear scene sequencer. Up to 64 rows of `[SCN | BARS | T1 T2 T3 T4 T5]`.

### Entering and exiting

Open the **SELECT** menu, then pick **song mode**. The transport stops, any queued scene clears, and the screen replaces the editor with the song table. The same menu row reads **exit song mode** while engaged; pick it to return.

Song mode is **exclusive**. While engaged, scene queueing and pattern launches are inert, and the topbar BPM, scale, and master controls are not reachable. Each row recalls the FX of its referenced scene, so editing BPM mid-song would conflict with that recall.

### Row anatomy

```
  ┌─────┬──────┬────┬────┬────┬────┬────┐
  │ SCN │ BARS │ T1 │ T2 │ T3 │ T4 │ T5 │
  └──┬──┴──┬───┴──┬─┴──┬─┴──┬─┴──┬─┴──┬─┘
     │     │      └────┴────┴────┴────┴── per-track pattern overrides
     │     └──────────────────────────── how many bars this row plays
     └────────────────────────────────── scene reference
```

- **SCN**: which scene this row references. Three states:
  - `N`: references scene N, with patterns matching scene N's defaults.
  - `N?`: references scene N, but at least one track's pattern has been overridden.
  - `?`: no scene reference. Patterns are authored directly; scale, transpose, and master inherit from the previously-fired scene in this song run.
- **BARS**: how long this row plays, `1x` to `99x` bars. `-` equals an empty row, which marks the end of the song.
- **T1..T5**: per-track pattern overrides. When you pick a scene in SCN, these auto-fill from that scene's patterns. Editing any T cell flips SCN to `N?`.

A row is essentially "play this scene for N bars, with optional per-track pattern overrides." The overrides exist so you can branch from a scene without authoring a whole new one. For example, take scene 2 but use T3's pattern 5 instead of its default pattern 1 for this row only.

### Editing

- **D-pad** moves the cursor between cells.
- **B + d-pad** edits the focused cell. On SCN and T cells, ±1 cycles values (any direction). On BARS, ↑/↓ equal ±10 bars (coarse), ←/→ equal ±1 bar (fine).
- **Y + d-pad** drags a row selection (vertical only; song rows are atomic).
- **A** cuts and pastes whole rows, same rules as the step grid.
- **SELECT** re-opens the menu.

Editing any non-BARS cell on an empty row (BARS = 0) auto-promotes BARS to **4** so the row becomes alive and visible immediately.

### Playback

- **Bare START** plays the song from the **cursor's current row**. Useful for auditioning a specific section without rewinding the song.
- **SELECT + START** forces playback to begin at **row 0**, regardless of cursor position.
- **START while playing** stops the transport.

Each row's scene fires at the start of its bar window; advance to the next row happens at the bar boundary after BARS bars elapse. Playback **stops** on the first empty row (BARS = 0).

---

## 14. Recording (master bounce)

Realtime capture of the post-master-bus stereo output to a WAV file.

### Start and stop

Open the **SELECT** menu, then pick **start recording**. The row flips to **stop recording** while a take is live. Pick it again to finalize.

Recording is **independent of transport**: REC arms capture immediately, regardless of whether the transport is playing. Tape-deck behavior. You can start REC, hit PLAY, stop PLAY, hit PLAY again, and the recording captures the whole thing across multiple transport cycles. STOP RECORDING finalizes the file.

### Output

- Files land in `/mnt/SDCARD/BRIXEQ/RECORDINGS/` on the SD card, named after the current project plus a date / time stamp.
- Format is standard 16-bit stereo WAV at 48 kHz. Plays in any audio app.
- Takes longer than 10 seconds are automatically gain-matched so quiet bounces don't come back as whispers.
- Takes shorter than 5 seconds are auto-deleted, so accidentally tapping REC twice doesn't litter the folder. A "RECORDING CANCELLED" toast confirms the file was discarded.
- Recordings are safe across power loss: even if the device loses power mid-record, the partial WAV on the SD card is still playable.
- The `BRIXEQ/` folder lives outside the pak directory, so PAK Store updates do not touch your recordings.

---

## 15. Save and load

Projects are saved as `.brixeq` files in `/mnt/SDCARD/BRIXEQ/PROJECTS/` on the SD card. The `BRIXEQ/` folder lives outside the pak directory, so PAK Store updates do not touch your saves.

### Save

SELECT menu, then **save**, then enter a name in the slot-cycling keyboard (12 char max, A-Z 0-9 - _).

A save captures **everything**: every step on every pattern on every track (including microtime), every scene, the song, master FX per scene, the active scene, BPM, shuffle. Audio recordings are kept separately in `/mnt/SDCARD/BRIXEQ/RECORDINGS/` and are not bundled inside project saves.

### Load

SELECT menu, then **load**, then pick from the list of saved projects. Loading replaces the in-memory project entirely. The transport stops on load.

### Auto-load

The last project saved or loaded is remembered and re-loaded automatically on the next boot.

---

## 16. Controls reference

| Button | Action |
|--------|--------|
| D-pad | Cursor navigation |
| **B** (south face) | Edit modifier. Hold plus d-pad to change the focused cell's value; tap alone to toggle trigless on a populated step |
| **A** (east face) | On a populated step: cut. On any other focused parameter (transpose cells, footer slots, BPM, scale, shuffle, master FX, modulator fields): reset to default. Menu confirm in popups |
| **Y** (west face) | Selection modifier. Hold plus d-pad to drag a selection rectangle |
| **X** (north face) | Column view cycle on the grid (steps → patterns → transpose). Works from any cursor region |
| **L1** | Focus left deck |
| **L2** | COPY focused step or selection into the clipboard (source untouched). Empty cells still copy, useful as a "wipe stamp" for R2 |
| **R1** | Focus right deck. Also clipboard modifier: **R1 + A** copies, **R1 + B** pastes (works for patterns, scenes, and master FX) |
| **R2** | OVERRIDE PASTE the clipboard at the cursor. Writes over populated cells; empty clipboard cells clear the target. Cursor advances past the pasted region |
| **L1 + L2** | UNDO the last edit. Walks the snapshot ring back one step |
| **R1 + R2** | REDO. Walks the snapshot ring forward one step |
| **L1 + R1 + B / Y / X / A** | RANDOMIZE the current step-grid selection. B = L1 density stamp, Y = L2 in-place mutation of the focused param on populated steps, X = L3 density+pitch+engine, A = L4 full per-engine. Footer reads `RANDOMIZE?` while both shoulders are held with a valid selection |
| **L3** | Toggle modulator screen |
| **R3** | Toggle master screen |
| **START** | Transport play / pause |
| **SELECT** | Open menu (new, save, load, record, song mode, help, quit). **SELECT + START** = play from beginning |

---

## 17. Tips and idioms

- **Pattern launches beat scene queues for live performance.** Scene queues replace everything (all five patterns plus scale plus transpose plus master FX). Pattern launches swap a single track's pattern. The latter is more granular and lets you build organic transitions.
- **Use scenes for harmonic motion.** Same patterns, different scale or transpose strip equals a fresh-sounding section without re-authoring 5 patterns.
- **Duplicate before you mutate.** R1+A copies a pattern or a scene; R1+B pastes it onto another slot. Spin a variation by cloning, then tweaking.
- **A button = reset.** On any focused parameter that isn't a step (transpose offsets, BPM, scale, shuffle, master cells, modulator fields), tapping A restores its default value. The fastest "undo my last few edits" gesture.
- **Modulators are per scene.** 8 LFOs that ride along on every scene swap. L3 to open, L3 again to leave. Try ONE + S&H on pitch for per-note random offsets, or FRE + slow sine on filter freq for a moving pad.
- **Master sends are envelope-gated.** A snare with a short decay sends a single sharp tap into the reverb; a held pad sustains the send open. Take advantage of this: different envelope shapes give wildly different bus-FX flavours.
- **Audition before you cut.** B-tap on the focused step plays its snapshot instantly. Pair with A (cut) and a cursor move + A (paste) to clone the exact instrument character to another step without waiting for the sequencer to come around.
- **Trigless steps morph the held note.** A row of trigless steps with sweeping filter freqs gives you continuous filter motion without retriggering. Same trick works for FM index, VA morph, drum body, pan, sends. The amp envelope keeps playing through — no clicks.
- **Double-tap A to copy.** Cut plus immediate paste-back equals source restored plus clipboard primed. Faster than a dedicated copy gesture.
- **Recording survives song mode.** You can arm REC before entering song mode and the entire song will be captured front to back.
- **Auto-load on boot picks up where you left off.** No need to hunt for the last project in the load list.
- **Randomize ladders well from sparse to dense.** Hold L1 + R1 on a selection, then tap B → B → B → B to fill 25 / 50 / 75 / 100 % of the cells with your last-copied step before any pitch / engine randomness kicks in. Switch to X / A when you want to layer extra mutation on top, the density cycle restarts each time you change level. Tap Y to mutate just the focused parameter (note, filter, chord, etc.) on the populated steps without touching their other params.

---

## 18. Troubleshooting

- **Occasional audio glitches.** If the audio crackles or stutters, reboot the device. BRIXEQ does not need a setup step on the Brick; the launcher ships with a buffer size that works on stock NextUI.
- **No audio at all.** Splash boots fine but no sequencer output. Check the device volume; also check that the headphone jack isn't toggling the internal speaker off.
- **Project won't load.** The save file may have been moved or modified outside BRIXEQ. The boot path gracefully falls back to a fresh project, and corrupted entries are hidden from the load list.
- **Cursor seems stuck.** You probably have a popup open (the SELECT menu, or the save / load dialog). Press SELECT or A to close it.
- **Step playhead doesn't advance.** Transport is stopped. Press START.
- **Transpose doesn't kick in.** Check the transpose strip's **length**. A length-1 strip is a no-op; bump it past 1 in transpose view's footer.

---

## 19. Glossary

- **Track**: one of five concurrent voices.
- **Pattern**: a 16-step sequence belonging to a track. Each track has 16 patterns.
- **Step**: one cell in a pattern. Carries a complete instrument snapshot.
- **Scene**: a snapshot of {one pattern per track, scale, transpose, master FX, modulators}. A project has 16.
- **Song**: a linear list of scene references (and overrides), up to 64 rows long.
- **Engine**: the synthesis algorithm for a step. Six available: VA, FM, NOIZ, BD, SD, CY.
- **Trigless**: a step that does NOT retrigger. Instead, when the playhead lands on it, its full params lock onto the currently-sounding voice on that track (filter, FM/VA/drum body, pan, level, sends). Envelopes keep their state; LFOs in TRIG/ONCE mode are not re-armed.
- **Chord stack**: root pitch plus up to 4 semitone offsets played together by the step.
- **Bar**: 16 master 16th-notes. The unit scene queues and song-row durations work in.
- **Master 16th**: one tick of the master clock, regardless of any track's clock divider.
- **Live launch**: a pattern swap on a single track, queued at the next bar.
- **Modulator**: a small per-scene LFO that nudges a chosen parameter on a chosen track over time. 8 slots per scene.
- **Randomize**: L1 + R1 + face button on a multi-cell step-grid selection. Four levels: B = density stamp from clipboard; Y = level 2, mutates only the focused parameter on populated steps in place; X = density + pitch + engine; A = full per-engine randomization. Same-level presses cycle density 25/50/75/100 → wrap. L2 same-(param) presses widen the ±N range 5/10/30/60 → wrap.
- **Run mode** (modulator): FRE = continuous; TRG = phase resets on each note; ONE = one cycle per note, then freezes.
- **S&H** (sample & hold): a waveform that holds a value for the cycle, then jumps to the next from a per-slot 16-value random table.

---
