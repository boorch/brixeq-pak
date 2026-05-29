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

- **Topbar**: project name (left), scale quantizer, BPM, and master-page icon (right). The scale and BPM cells are "editable"; the master icon opens the master FX page.
- **Track row**: for each of the five tracks, a track header (mute, spectrum), pattern slots (the 16-pattern bank as a vertical strip), the **step grid** (the 16-step pattern), and the **track footer** (length, direction, clock-div for the focused pattern).
- **Left and right decks**: when a step is focused, both sides slide on screen to show parameter cells. The **left deck** carries engine-common params (pitch, logic, envelope, filter, output and sends). The **right deck** is engine-specific (different controls for VA vs FM vs the drum engines).
- **Scenes strip**: sixteen scene cells along the bottom, just above the footer.
- **Footer**: transient toasts, mode label, hints.

Pressing **X** on the step grid cycles between three column views: **steps → patterns → transpose**. Steps view is the default; the others let you edit the pattern bank or the per-track transpose sequence inline without leaving the main editor.

---

## 5. Cursor and navigation

- D-pad **moves the cursor**. The cursor is one-cell, it lives on a specific track and column.
- **L1 / R1** focus the left or right deck respectively. The cursor jumps into the deck's currently-active row; releasing returns it to the step grid.
- **Topbar** is reached by pressing **↑** while on the top step of any track; the cursor climbs out of the grid into the scale, BPM, and master cells.
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

A trigless step holds a slot in the sequence (so the cursor can land on it) but won't fire during normal playback, useful for placeholders or rests.

### Editing topbar cells

You can edit the **scale** and **BPM** cells in the topbar the same way as a step: navigate the cursor up into the topbar, focus the cell, then B + d-pad.

### Selection

Hold **Y** while moving the d-pad to drag a selection rectangle from the cursor. The cursor follows the far corner of the selection. Releasing Y leaves the selection in place; pressing Y again extends it from where you left off.

Multi-tap behavior on Y:
- One tap (no movement): clear selection.
- Two taps quickly: select the whole track at the cursor column.
- Three taps: select the entire grid.

Pressing any bare d-pad arrow clears the selection and resumes single-cell cursor navigation.

### Cut / copy / paste

A single button, **A**, does both, deciding by context:

- **A on a populated step**: **cut**. The step is captured into the clipboard and cleared.
- **A on an empty step with clipboard primed**: **paste**.
- **A with a multi-cell selection that contains anything populated**: multi-cut.
- **A with a multi-cell selection that is all empty plus clipboard primed**: multi-paste at the selection's top-left.

Double-tap A on the same populated step equals cut plus paste-back, which equals **copy** (the source is restored, the clipboard stays primed for stamping elsewhere). The same idiom works on multi-cell selections.

The clipboard captures the **whole step snapshot** (engine, every param, chord, logic, the lot), so a paste recreates the source step's instrument character exactly.

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
- **RULE**: logic (trigger conditions, e.g. every nth bar, alternating) and probability (1/4, 1/2, 3/4, 1/1).
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

---

## 8. Engines

Six engines, selectable per-step:

- **VA**: virtual analog. Three timbre macros (harmonics, timbre, morph) blend continuously between several oscillator characters. Plus overdrive. Pitched.
- **FM**: two-operator FM. Modulator ratio, depth, feedback, attack, decay. Carrier is a fixed sine. Pitched, percussive bite.
- **NOIZ**: noise generator. Several noise types (narrow, white, pink, brown) cover the range from tonal hiss to dark rumble. The lowpass and highpass on the left deck shape it. Mostly unpitched (the note value tunes the narrow noise modes).
- **BD**: 808/909-style bass drum.
- **SD**: 808/909-style snare drum.
- **CY**: hi-hat / cymbal. Covers both sounds via the timbre and morph controls.

Drum engines **ignore transpose** and **scale quantization**. Transposing a drum note would just remap which drum sample plays, which isn't useful.

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

**R1 + A** copies the active scene's master FX block to a clipboard. **R1 + B** pastes it onto whichever scene cell is currently focused in the scenes strip. Lets you author one master sound and reuse it across scenes without re-authoring every parameter.

---

## 13. Song mode

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

- Files land in the `RECORDINGS` folder on the SD card, named after the current project plus a date / time stamp.
- Format is standard 16-bit stereo WAV at 48 kHz. Plays in any audio app.
- Takes longer than 10 seconds are automatically gain-matched so quiet bounces don't come back as whispers.
- Takes shorter than 5 seconds are auto-deleted, so accidentally tapping REC twice doesn't litter the folder. A "RECORDING CANCELLED" toast confirms the file was discarded.
- Recordings are safe across power loss: even if the device loses power mid-record, the partial WAV on the SD card is still playable.

---

## 15. Save and load

Projects are saved as `.brixeq` files in the `PROJECTS` folder on the SD card.

### Save

SELECT menu, then **save**, then enter a name in the slot-cycling keyboard (12 char max, A-Z 0-9 - _).

A save captures **everything**: every step on every pattern on every track, every scene, the song, master FX per scene, the active scene, BPM. Audio recordings are kept separately in the `RECORDINGS` folder and are not bundled inside project saves.

### Load

SELECT menu, then **load**, then pick from the list of saved projects. Loading replaces the in-memory project entirely. The transport stops on load.

### Auto-load

The last project saved or loaded is remembered and re-loaded automatically on the next boot.

---

## 16. Controls reference

| Button | Action |
|--------|--------|
| D-pad | Cursor navigation |
| **B** (south face) | Edit modifier. Hold plus d-pad to change the focused cell's value; tap alone to toggle trigless |
| **A** (east face) | Confirm / cut / paste. Single action button on steps; menu confirm in popups |
| **Y** (west face) | Selection modifier. Hold plus d-pad to drag a selection rectangle |
| **X** (north face) | Column view cycle on the grid (steps → patterns → transpose) |
| **L1** | Focus left deck |
| **L2** | Reserved |
| **R1** | Focus right deck. Also master copy modifier on the master page |
| **R2** | Reserved |
| **START** | Transport play / pause |
| **SELECT** | Open menu (save, load, record, song mode, quit) |

---

## 17. Tips and idioms

- **Pattern launches beat scene queues for live performance.** Scene queues replace everything (all five patterns plus scale plus transpose plus master FX). Pattern launches swap a single track's pattern. The latter is more granular and lets you build organic transitions.
- **Use scenes for harmonic motion.** Same patterns, different scale or transpose strip equals a fresh-sounding section without re-authoring 5 patterns.
- **Master sends are envelope-gated.** A snare with a short decay sends a single sharp tap into the reverb; a held pad sustains the send open. Take advantage of this: different envelope shapes give wildly different bus-FX flavours.
- **Audition before you cut.** B-tap on the focused step plays its snapshot instantly. Pair with A (cut) and a cursor move + A (paste) to clone the exact instrument character to another step without waiting for the sequencer to come around.
- **Double-tap A to copy.** Cut plus immediate paste-back equals source restored plus clipboard primed. Faster than a dedicated copy gesture.
- **Recording survives song mode.** You can arm REC before entering song mode and the entire song will be captured front to back.
- **Auto-load on boot picks up where you left off.** No need to hunt for the last project in the load list.

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
- **Scene**: a snapshot of {one pattern per track, scale, transpose, master FX}. A project has 16.
- **Song**: a linear list of scene references (and overrides), up to 64 rows long.
- **Engine**: the synthesis algorithm for a step. Six available: VA, FM, NOIZ, BD, SD, CY.
- **Trigless**: a step that holds its slot in the grid but doesn't fire.
- **Chord stack**: root pitch plus up to 4 semitone offsets played together by the step.
- **Bar**: 16 master 16th-notes. The unit scene queues and song-row durations work in.
- **Master 16th**: one tick of the master clock, regardless of any track's clock divider.
- **Live launch**: a pattern swap on a single track, queued at the next bar.

---
