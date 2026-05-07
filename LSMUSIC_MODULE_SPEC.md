# ButterflyDreaming L-System Music Module — Specification v0.1

## Overview

This document is the source of truth for the L-system music module for the
ButterflyDreaming platform. It describes two HTML files — `index.html` (harness)
and `lsmusic_module.html` (the module). Claude Code should read this document in
full before writing any code, and should also read `VISUAL_MODULE_SPEC.md` and
`MUSIC_MODULE_SPEC.md` for inherited patterns.

---

## Context

ButterflyDreaming is a dyadic encounter platform where two anonymous users
collaboratively explore and weave symbolic text nodes. The platform is documented
at https://butterflydreaming.info.

Media modules are self-contained HTML files that can be triggered from within the
platform to render a text node as sound, image, or other media. They communicate
with the main platform via the browser postMessage API. The text node content is
the single source of truth — all rendering parameters are encoded within it.

This module implements the music generation approach described in Prusinkiewicz &
Hanan, *Lindenmayer Systems, Fractals, and Plants* (Springer, 1989), section 6.3.
An L-system grammar is iterated to produce a fractal curve, which is then
interpreted as a musical score using turtle geometry. The generated ABC notation
is an internal intermediate product — it is not displayed in the UI but can be
copied to the clipboard.

The L-system grammar input, depth control, and `%%bd_` directive framework are
inherited from the 2D visual module (`butterflydreaming_visual_1`). The audio
rendering pipeline, effects sliders (reverb wet, reverb decay, vibrato frequency,
vibrato depth, chorus wet, chorus depth), loop controls, and sampler setup are
inherited from the music module (`butterflydreaming_music_1`).

---

## Project Structure

```
butterflydreaming_lsmusic_1/
  LSMUSIC_MODULE_SPEC.md      ← this file
  index.html                  ← harness (simulates the BD platform)
  lsmusic_module.html         ← the media module
  bass-recorder/              ← sample files (same as music module)
    bass_recorder_F#2.mp3
    bass_recorder_C3.mp3
    bass_recorder_G#3.mp3
    bass_recorder_E4.mp3
```

---

## Libraries

CDN only, no build step.

| Library      | Purpose                          | CDN |
|--------------|----------------------------------|-----|
| Tone.js      | Audio engine and Sampler         | https://cdnjs.cloudflare.com/ajax/libs/tone/14.7.77/Tone.js |
| abcjs        | ABC notation parser              | https://cdnjs.cloudflare.com/ajax/libs/abcjs/6.2.2/abcjs-basic-min.js |
| lindenmayer  | L-system iteration               | https://cdn.jsdelivr.net/npm/lindenmayer@2.0.0/src/lindenmayer.js |

---

## BD Messaging Protocol

Identical to the visual and music modules. See `VISUAL_MODULE_SPEC.md` section
"The postMessage Protocol". Messages are:

- `BD_READY` — module → harness on load
- `BD_INIT` — harness → module, carries node text
- `BD_UPDATE` — module → harness, carries updated node text
- `BD_REQUEST_UPDATE` — harness → module, requests current state
- `BD_STOP` — harness → module
- `BD_ERROR` — module → harness on failure

Origin check uses `"*"` during development. Tighten to
`"https://butterflydreaming.info"` in production.

---

## The `%%bd_` Directive Format

Follows Amendment 20 of `MUSIC_MODULE_SPEC.md` exactly. Single-line directives
terminated by newline, no semicolons. The directive syntax is:

```
%%bd_module lsmusic_module.html
%%bd_axiom X
%%bd_rules [
X=-YF+XFX+FY-
Y=+XF-YFY-FX+
%%bd_]
%%bd_depth 4
%%bd_angle 90
%%bd_scale pentatonic
%%bd_tempo 80
%%bd_octave_centre 4
%%bd_loop true
%%bd_loop_gap 4
%%bd_reverb_wet 0.35
%%bd_reverb_decay 2.5
%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.2
%%bd_chorus_wet 0.3
%%bd_chorus_depth 0.4
```

The `%%bd_rules` block uses the multi-line bracket syntax from Amendment 20 of
`MUSIC_MODULE_SPEC.md`. Each line inside the block is a single production rule
in the format `symbol=replacement`.

---

## L-System Grammar

### Default Grammar — Hilbert Curve

From Prusinkiewicz & Hanan, *Lindenmayer Systems, Fractals, and Plants*,
Figure 2.7(b):

```
Axiom: X
X → −YF+XFX+FY−
Y → +XF−YFY−FX+
Angle: 90°
```

This produces a space-filling curve with clean horizontal and vertical segments,
which maps unambiguously to the Prusinkiewicz pitch/duration model.

### Iteration

Use the lindenmayer library identically to the visual module. Depth range 1–5.
At depth 4 the Hilbert curve produces approximately 340 horizontal segments —
musically rich but schedulable. At depth 5 the note count exceeds 1300 and may
cause noticeable scheduling delay. Depth 4 is the default.

---

## Musical Interpretation

This section implements the Prusinkiewicz method (section 6.3, *Lindenmayer
Systems, Fractals, and Plants*).

### Which Symbols Produce Notes

Only `F` (forward draw) produces a note. All other symbols (`+`, `−`, `X`, `Y`,
and any other non-drawing symbols) are silently ignored during musical
interpretation.

### Turtle Geometry

Run a standard 2D turtle interpreter over the iterated string:

- Initial heading: 0° (pointing right, positive X direction)
- Initial position: (0, 0)
- Step length: 1 unit
- `F` — move forward one step, record start and end position
- `+` — turn left by angle
- `−` — turn right by angle
- All other symbols — no action

At 90° all segments are strictly horizontal or vertical.

### Segment Classification

After turtle interpretation, classify each `F` segment:

- **Horizontal segment** — start Y and end Y are equal within floating point
  tolerance of 0.001. This segment **produces a note**.
- **Vertical segment** — start X and end X are equal. This segment produces
  **no note** and is silently skipped.

### Pitch Mapping

The Y-coordinate of a horizontal segment determines its pitch.

**Octave centre:** `%%bd_octave_centre` sets the central octave (default 4,
i.e. middle C octave). Y=0 maps to the tonic of the central octave.

**Y normalisation:** Compute the min and max Y across all horizontal segments.
Map linearly across this range to ±2 octaves from the octave centre. Clamp to
the sampler's usable range (F#2 to E4 — the bass recorder sample range). The
Y range is computed fresh on each recompute, so the pitch mapping is always
relative to the current grammar and depth.

**Scale lookup tables:**

```javascript
const scales = {
  major:      [0, 2, 4, 5, 7, 9, 11],   // C D E F G A B
  pentatonic: [0, 2, 4, 7, 9],           // C D E G A
  blues:      [0, 3, 5, 6, 7, 10]        // C Eb F F# G Bb
};
```

Map the normalised Y value to a scale degree index, then to a Tone.js pitch
string (e.g. `"C4"`, `"G3"`). The root is always C in v0.1. Key transposition
is a future amendment.

### Duration Mapping

All `F` segments have length 1 (unit step), therefore all notes produced by
horizontal segments have equal duration. The base note duration in seconds is:

```
note_duration_seconds = 60 / %%bd_tempo
```

At the default 80 BPM each note is 0.75 seconds.

### Rest Handling

Vertical segments produce no note. Silence between notes occurs naturally via
the Tone.js sampler's decay — no explicit rests need to be inserted.

---

## ABC Generation (Internal)

After interpreting the turtle output, generate an ABC string for internal use
and for the Copy Music button. This ABC is never displayed in the UI.

```
X:1
T:L-System Fragment
M:4/4
L:1/8
Q:1/4=[tempo]
K:C
```

Group notes into bars of 4/4. Each note is expressed in ABC pitch notation.
Barlines are inserted automatically every 4 beats. The ABC is passed to abcjs
for scheduling and to Tone.js for playback, exactly as in the music module.

---

## Audio Rendering

Identical to the music module (`MUSIC_MODULE_SPEC.md`). Use the same Tone.js
Sampler with the four bass recorder samples. Apply the same effects chain:
reverb, vibrato, chorus, controlled by the same `%%bd_` directives. Looping
is controlled by `%%bd_loop` and `%%bd_loop_gap`.

---

## Control Change Behaviour

Changes to depth, scale, tempo, and all audio effects sliders are registered
immediately in the UI but take effect at the next loop boundary — after the
current playback phrase completes and before the loop gap silence begins.
At that point the module:

1. Re-iterates the L-system to the new depth if depth has changed
2. Re-runs the turtle interpreter and segment classification
3. Regenerates the ABC internally
4. Reschedules the Tone.js sequence with any new scale, tempo, or effects values
5. Resumes playback after the loop gap

If playback is not currently looping, changes take effect on the next manual
Play press.

---

## File 1: index.html (Harness)

### Purpose

Simulates the BD platform for development and testing. Not part of the final
platform.

### Layout

Follows the visual module harness pattern. Dark background, two columns:

**Left column — Controls:**
- Label: `EXTENDED L-SYSTEM NOTATION`
- A `<textarea>` (id: `ls-input`) pre-populated with the default node text
  (see Default Node Text below)
- Button: `Send to Player` — posts `BD_INIT` to the iframe
- Button: `Request Update` — posts `BD_REQUEST_UPDATE` to the iframe
- Status area showing last message received from module

**Right column — Player:**
- An `<iframe>` (id: `lsmusic-module`) pointing to `lsmusic_module.html`
- 600×500px

### Default Node Text

```
%%bd_module lsmusic_module.html
%%bd_axiom X
%%bd_rules [
X=-YF+XFX+FY-
Y=+XF-YFY-FX+
%%bd_]
%%bd_depth 4
%%bd_angle 90
%%bd_scale pentatonic
%%bd_tempo 80
%%bd_octave_centre 4
%%bd_loop true
%%bd_loop_gap 4
%%bd_reverb_wet 0.35
%%bd_reverb_decay 2.5
%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.2
%%bd_chorus_wet 0.3
%%bd_chorus_depth 0.4
```

---

## File 2: lsmusic_module.html (The Module)

### Layout

Dark background, calm minimal aesthetic consistent with the visual and music
modules.

**Top row — Status:**
- Status line: e.g. `Loading samples...` / `Ready` / `Playing` / `Stopped`
- Small label: `Module: lsmusic_module.html`

**Controls row:**
- `Play` button (disabled until samples loaded and grammar parsed)
- `Stop` button
- `Copy Music` button — copies generated ABC to clipboard; disabled until first
  successful parse; briefly shows `Copied!` after copy then reverts

**Grammar controls:**
- Depth selector: dropdown 1–5, labelled `DEPTH`
- Scale selector: dropdown with options `major` / `pentatonic` / `blues`,
  labelled `SCALE`
- Tempo slider: range 40–160 BPM, labelled `TEMPO`, displays current BPM value

**Grammar area:**
- Label: `L-SYSTEM GRAMMAR`
- Axiom input: single-line text field, labelled `AXIOM`
- Rules textarea: multi-line, labelled `RULES` (one rule per line,
  format `X=replacement`)

**Audio effects sliders** (inherited from music module):
- Reverb Wet (0.0–1.0)
- Reverb Decay (0.1–10.0 seconds)
- Vibrato Frequency (1.0–10.0 Hz)
- Vibrato Depth (0.0–1.0)
- Chorus Wet (0.0–1.0)
- Chorus Depth (0.0–1.0)

**No ABC display.** The generated ABC is internal only.

### Initialisation Sequence

1. Post `BD_READY` to parent immediately on load
2. Begin loading Tone.js Sampler with the four bass recorder samples
3. Set status to `Loading samples...`
4. Listen for postMessage events

### Message Handling

**BD_INIT received:**
1. Parse all `%%bd_` directives from the node text
2. Populate axiom field, rules textarea, depth, scale, tempo, and all audio
   effects controls from directives
3. Iterate the L-system to the specified depth using the lindenmayer library
4. Run turtle interpreter and classify segments
5. Generate ABC internally
6. Schedule Tone.js sequence
7. If any step fails post `BD_ERROR` and update status
8. If successful set status to `Ready — press Play`
9. Do not autoplay

**BD_REQUEST_UPDATE received:**
Post `BD_UPDATE` with the current node text reconstructed from current control
values — grammar directives, depth, scale, tempo, octave centre, loop settings,
and all audio effects directives. The generated ABC is **not** included in the
returned text.

**BD_STOP received:**
Stop playback immediately, set status to `Stopped`.

---

## Known Limitations (v0.1)

- Root key is always C — transposition is a future amendment
- Only horizontal segments produce notes — non-right-angle grammars are a
  future amendment using complex number projection
- Single production rule format only (`symbol=replacement`) — stochastic and
  parametric L-systems are future work
- No visual rendering — a combined visual+music module is a future amendment

---

## Amendment Record

*Amendments to be recorded here as they are made, following the same convention
as `VISUAL_MODULE_SPEC.md` and `MUSIC_MODULE_SPEC.md`.*
