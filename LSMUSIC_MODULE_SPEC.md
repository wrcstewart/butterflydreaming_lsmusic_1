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

### Default Grammar — Koch Island

The default single-voice grammar. Uses only the Voice 1 symbol set.

```
Axiom: F-F-F-F
F=F+FF-FF-F-F+F+FF-F+F-F-FF+FF+
Angle: 90°
Depth: 1
```

### Iteration

Use the lindenmayer library identically to the visual module. Depth range 1–5.

---

## Musical Interpretation

This section implements the Prusinkiewicz method (section 6.3, *Lindenmayer
Systems, Fractals, and Plants*).

### Voice 1 Symbol Set

| Symbol | Role |
|--------|------|
| `F` | Forward step — produces a note if horizontal |
| `+` | Turn left by angle |
| `-` | Turn right by angle |

All other symbols are transparent to Turtle 1 and ignored during interpretation.

### Voice 2 Symbol Set

| Symbol | Role | Voice 1 equivalent |
|--------|------|--------------------|
| `G` | Forward step — produces a note if horizontal | `F` |
| `L` | Turn left by angle | `+` |
| `R` | Turn right by angle | `-` |

All other symbols are transparent to Turtle 2 and ignored during interpretation.

### Turtle Geometry

Run a standard 2D turtle interpreter over the iterated string:

- Initial heading: 0° (pointing right, positive X direction)
- Initial position: (0, 0)
- Step length: 1 unit
- Forward symbol — move forward one step, record start and end position
- Left turn symbol — turn left by angle
- Right turn symbol — turn right by angle
- All other symbols — no action (transparent)

At 90° all segments are strictly horizontal or vertical.

### Segment Classification

After turtle interpretation, classify each forward step:

- **Horizontal segment** — start Y and end Y are equal within floating point
  tolerance of 0.001. This segment **produces a note**.
- **Vertical segment** — produces no note and is silently skipped.

### Pitch Mapping

The Y-coordinate of a horizontal segment determines its pitch.

**Y normalisation:** Compute the min and max Y across all horizontal segments.
Map linearly across this range to the sampler's usable MIDI range (F#2 to E4,
MIDI 42–64). Clamp all values to this range. The Y range is computed fresh on
each recompute per voice independently.

**`%%bd_octave_centre`:** Currently reserved — not functionally implemented.
The pitch mapping uses the full sampler range regardless of this value.
Proper implementation is a future amendment.

**Scale lookup tables:**

```javascript
const scales = {
  major:      [0, 2, 4, 5, 7, 9, 11],   // C D E F G A B
  pentatonic: [0, 2, 4, 7, 9],           // C D E G A
  blues:      [0, 3, 5, 6, 7, 10]        // C Eb F F# G Bb
};
```

Map each MIDI value to the nearest in-scale pitch. The root is always C in v0.1.

### Duration Mapping

The duration of each note is proportional to its run length (Amendment 6):

```
note_duration_seconds = run_length × (60 / %%bd_tempo)
```

### Same-Pitch Run Merging

After pitch mapping, consecutive notes of identical pitch are merged into a
single note of combined duration (Amendment 6). This applies independently
to each voice.

### Rest Handling

Vertical segments produce no note. Silence occurs naturally via the Tone.js
sampler's decay.

---

## ABC Generation (Internal)

After interpreting the turtle output, generate an ABC string for internal use
and for the Copy ABC Score button. This ABC is never displayed in the UI.

```
X:1
T:L-System Fragment — Voice 1
M:4/4
L:1/8
Q:1/8=[tempo]
K:C
```

Group notes into bars of 4/4. Each note is expressed in ABC pitch notation with
correct length suffix (e.g. `C2`, `G4`). Barlines are inserted automatically
every 8 eighth-note units. Notes longer than the remaining bar space are tied
across the barline.

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
- Button: `Send to Player` — posts `BD_INIT` to the iframe
- Button: `Request Update` — posts `BD_REQUEST_UPDATE` to the iframe
- Three copy buttons below the textarea:
  - `COPY ABC SCORE` — copies GENERATED ABC SCORE text area content
  - `COPY SCRIPT` — copies script textarea content
  - `COPY LINK` — encodes script as URL and copies to clipboard
- Label: `GENERATED ABC SCORE`
- Read-only scrollable text area showing generated ABC (both voices if present)
- Status area showing last message received from module

**Right column — Player:**
- An `<iframe>` (id: `lsmusic-module`) pointing to `lsmusic_module.html`
- 600×500px

### Default Node Text

```
%%bd_module lsmusic_module.html
%%bd_loop true
%%bd_loop_gap 4
%%bd_reverb_wet 0.35
%%bd_reverb_decay 2.5
%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.2
%%bd_chorus_wet 0.3
%%bd_chorus_depth 0.4
%%bd_tempo 80
%%bd_octave_centre 4

%%bd_voice 1
%%bd_scale pentatonic
%%bd_depth 1
%%bd_angle 90
%%bd_axiom F-F-F-F
%%bd_rules [
F=F+FF-FF-F-F+F+FF-F+F-F-FF+FF+
%%bd_]

%%bd_voice 2
%%bd_scale pentatonic
%%bd_depth 1
%%bd_angle 90
%%bd_axiom GRGRGRG
%%bd_rules [
G=GLGGRGGRGRGLGLGGRGLGRGRGGLGGL
%%bd_]
```

### URL Parameter Loading

On page load the correct sequence must be written as a single linear flow:

```javascript
// CORRECT sequence — do not reorder
const scriptParam = new URLSearchParams(window.location.search).get('script');
if (scriptParam) {
    textarea.value = scriptParam;   // URLSearchParams already decoded it
} else {
    textarea.value = DEFAULT_SCRIPT;
}
sendToPlayer();   // populate player but do NOT autoplay
```

> ⚠ **Known sequencing pitfall** — also encountered in the 2D visual module
> (VISUAL_MODULE_SPEC.md Amendment 12). The URL parameter check must execute
> before any default script is applied or any auto-send is triggered.

### Copy Link Encoding

```javascript
// Encode (Copy Link button)
const url = `https://wrcstewart.github.io/butterflydreaming_lsmusic_1/?script=${encodeURIComponent(textarea.value)}`;

// Decode (page load) — URLSearchParams.get() already decodes, no further call needed
const scriptParam = new URLSearchParams(window.location.search).get('script');
```

> ⚠ Do NOT use `btoa`/`atob` for encoding — base64 contains `+` which
> `URLSearchParams` treats as a space, corrupting the round-trip. Use
> `encodeURIComponent` only.

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
- Depth selector: dropdown 1–5, labelled `DEPTH`
- Scale selector: dropdown `major` / `pentatonic` / `blues`, labelled `SCALE`
- Tempo slider: range 40–160 BPM, labelled `TEMPO`, displays current BPM value

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
1. Parse all `%%bd_` directives from the node text — shared header then per-voice blocks
2. Populate controls from directives
3. For each voice: iterate L-system, run turtle interpreter, classify segments,
   generate ABC internally, schedule Tone.js sequence
4. If any step fails post `BD_ERROR` and update status
5. If successful set status to `Ready — press Play`
6. Do not autoplay
7. Post `BD_UPDATE` immediately with `generatedABC` field for debug display

**BD_REQUEST_UPDATE received:**
Read current values from all live controls. Reconstruct full two-voice script —
shared header directives first, then each voice block with current axiom, rules,
depth, angle, scale. Post `BD_UPDATE` with reconstructed text and `generatedABC`.
Generated ABC is not included in the node text itself.

**BD_STOP received:**
Stop playback immediately, set status to `Stopped`.

---

## Two-Voice Architecture

### Script Structure

The node text is divided into a shared header block followed by one or two voice
blocks. The `%%bd_voice` directive opens a new voice block. All directives before
the first `%%bd_voice` directive are shared.

**Backwards compatibility:** A script with no `%%bd_voice` directive is treated
as a single Voice 1 script. Voice 2 is silent. All existing single-voice scripts
work without modification.

### Directive Parsing

**Pass 1 — shared directives:** All directives before the first `%%bd_voice`
line. These apply to both voices.

**Pass 2 — voice blocks:** From each `%%bd_voice` line until the next
`%%bd_voice` line or end of script.

Per-voice: `%%bd_scale`, `%%bd_depth`, `%%bd_angle`, `%%bd_axiom`, `%%bd_rules`

Shared: `%%bd_tempo`, `%%bd_reverb_*`, `%%bd_vibrato_*`, `%%bd_chorus_*`,
`%%bd_loop`, `%%bd_loop_gap`, `%%bd_octave_centre`

### Slider Override Behaviour

Slider values override both voices simultaneously. Script values are the initial
state on BD_INIT; sliders take live control from that point:

- Scale slider — overrides scale for both voices
- Tempo slider — overrides tempo for both voices
- All effects sliders — apply to both sampler instances
- Depth selector — overrides depth for both voices

Changes take effect at the next loop boundary.

### Two-Turtle Rendering

Each voice's axiom and rules are iterated independently by the lindenmayer
library. The resulting string is interpreted by the appropriate turtle:

- **Turtle 1** acts on `F`, `+`, `-` — every other symbol is transparent
- **Turtle 2** acts on `G`, `L`, `R` — every other symbol is transparent

Both turtles start at (0, 0) heading East (0°). Each maintains its own x, y,
angle state independently.

**Empty voice handling:** If a voice produces no horizontal segments it is
silently ignored. No error is raised.

### Two Separate ABC Scores

Generate two completely independent ABC strings. Two independent Tone.js Part
instances on the shared Tone.Transport play simultaneously. Loop boundary is the
longer of the two sequence durations — the shorter voice is silent for the
remainder.

### Future Architecture Note — Mixed Symbol Sets

In future amendments production rules will be allowed to mix the two symbol
alphabets freely. When this is introduced the two voices will share a single
combined iterated string. The two-turtle model handles this automatically:

- **Turtle 1** iterates the full combined string ignoring everything except
  `F`, `+`, `-`
- **Turtle 2** iterates the full combined string ignoring everything except
  `G`, `L`, `R`

No change to the rendering pipeline is required — only the script parsing and
L-system iteration step changes.

---

## Known Limitations (v0.1)

- Root key is always C — transposition is a future amendment
- `%%bd_octave_centre` is not functionally implemented — future amendment
- Only horizontal segments produce notes — non-right-angle grammars are a
  future amendment using complex number projection
- Both voices use the same sampler timbre — independent timbres are a future
  amendment
- Rule intermixing between voices (shared combined string) is a future amendment
- No visual rendering — combined visual and music module is a future amendment

---

## Amendment Record

### Amendment 1 — Correct Duration Mapping to Prusinkiewicz Method

The initial implementation treated every horizontal `F` segment as a note of
equal duration. The Prusinkiewicz method (section 6.3, Figure 6.9) states that
note duration is proportional to segment length — the count of consecutive
horizontal `F` steps forming a continuous horizontal run.

#### Corrected Segment Classification

Accumulate consecutive horizontal `F` steps into runs. A run ends when:
- A turn symbol is encountered
- A vertical `F` step occurs
- The string is exhausted

Each completed horizontal run produces exactly one note.

#### Corrected Duration Mapping

```
note_duration_seconds = run_length × (60 / %%bd_tempo)
```

A run of length `n` produces a note expressed as `Xn` in ABC (e.g. `C2`, `G4`).

---

### Amendment 2 — ABC Debug Display and Note Duration Fix

#### Part A — ABC Debug Display

Add a read-only text area below the script input box in `index.html` labelled
`GENERATED ABC`. Populated every time the module posts a `BD_UPDATE` message
containing a `generatedABC` field. Read-only but selectable and scrollable.

In `lsmusic_module.html`, include the generated ABC string as a `generatedABC`
field in every `BD_UPDATE` message payload.

#### Part B — Note Duration Fix

The run-length accumulation must operate at the ABC generation stage, not at
the Tone.js scheduling stage.

---

### Amendment 3 — Fix ABC Debug Display Population

After a successful `BD_INIT` sequence, immediately post a `BD_UPDATE` message
to the parent containing the `generatedABC` field. Do not wait for
`BD_REQUEST_UPDATE`.

```javascript
window.parent.postMessage({
  type: 'BD_UPDATE',
  text: currentNodeText,
  generatedABC: currentABC
}, '*');
```

---

### Amendment 4 — Implement ABC Run-Length Accumulation Correctly

The ABC generator must implement run-length accumulation as specified in
Amendment 1. The `consecutive` flag on each horizontal segment must be set
during turtle interpretation:

- `true` if the immediately preceding symbol was also a horizontal forward step
- `false` if the preceding symbol was a turn, a vertical step, or start of string

Run accumulation:

```javascript
let i = 0;
while (i < horizontalSegments.length) {
    let run = 1;
    while (i + run < horizontalSegments.length &&
           horizontalSegments[i + run].consecutive === true) {
        run++;
    }
    abc += horizontalSegments[i].pitch + (run > 1 ? run : '') + ' ';
    i += run;
}
```

---

### Amendment 5 — Fix Run-Break Logic

A run is broken **only** by:
- A turn symbol (`+` or `-` for Voice 1; `L` or `R` for Voice 2)
- A vertical forward step

Non-drawing symbols are **transparent** to run accumulation and must not reset
the run counter. The reset condition must be:

```javascript
// CORRECT — only turns break a run
if (symbol === '+' || symbol === '-') prevHorizF = false;
// NOT: if (symbol !== 'F') prevHorizF = false;
```

---

### Amendment 6 — Same-Pitch Note Merging and Pitch Mapping Fix

#### Part A — Pitch Mapping Fix

The original implementation mapped the Y range to 48 semitones (4 octaves),
causing notes in the upper half of the range to clamp to E4. The fix maps
`[yMin, yMax]` linearly to `[MIDI_MIN, MIDI_MAX]` (42–64) and snaps each
value to the nearest in-scale MIDI note.

#### Part B — Same-Pitch Note Merging

After pitch mapping, add a post-processing pass that merges consecutive notes
of identical pitch into a single note with combined duration:

```javascript
function mergeConsecutivePitches(notes) {
  if (notes.length === 0) return [];
  const merged = [];
  for (const note of notes) {
    if (merged.length > 0 &&
        merged[merged.length - 1].pitch === note.pitch) {
      merged[merged.length - 1].runLength += note.runLength;
    } else {
      merged.push({ ...note });
    }
  }
  return merged;
}
```

This pass runs **after** `mapYToPitch()` and **before** `buildABC()`.

---

### Amendment 7 — Fix BD_REQUEST_UPDATE Slider Values

The `BD_REQUEST_UPDATE` handler was reading stored parsed directive values
rather than live control state. Fix: read current values directly from every
control element at the moment the request is received.

---

### Amendment 8 — Debug Label, Copy Buttons and URL Parameter Loading

#### 8.1 — Rename Debug Label
Change label from `GENERATED ABC` to `GENERATED ABC SCORE`.

#### 8.2 — Add Three Copy Buttons

**COPY ABC SCORE** — copies GENERATED ABC SCORE content. Disabled if empty.
Briefly shows `Copied ✓` for 1.5 seconds.

**COPY SCRIPT** — copies script textarea content.
Briefly shows `Copied ✓` for 1.5 seconds.

**COPY LINK** — encodes script with `encodeURIComponent`, constructs URL:
`https://wrcstewart.github.io/butterflydreaming_lsmusic_1/?script=ENCODED`
Briefly shows `Link Copied ✓` for 1.5 seconds.

#### 8.3 — URL Parameter Loading and Copy Link Encoding

See URL Parameter Loading and Copy Link Encoding sections in File 1: index.html
above. The key constraint: use `encodeURIComponent` only — never `btoa`/`atob`.

> ⚠ Known sequencing pitfall: URL check must run before default script load.
> See VISUAL_MODULE_SPEC.md Amendment 12.

---

### Amendment 9 — Two-Voice System

See Two-Voice Architecture section above, which incorporates all corrections
from Amendments 10 and 11. The Voice 2 symbol set uses `G`, `L`, `R`
throughout (not `<` and `>`).

---

### Amendment 10 — Voice 2 Symbol Change and Pitch Mapping Fix

#### 10.1 — Replace `<`/`>` with `L`/`R`

The symbols `<` and `>` are replaced with `L` (turn left) and `R` (turn right)
for Voice 2. This avoids HTML rendering issues at end of lines.

Turtle 2 now responds to `G`, `L`, `R`. Update all grammars accordingly.

#### 10.2 — Pitch Mapping Inconsistency Fix

The unison test revealed different ABC output for geometrically identical
segment sequences. Root cause: `mapYToPitch()` was not being called identically
for both voices. Fix: confirm `yMin`/`yMax` are computed separately from each
voice's own segment array, same scale lookup and octave centre used for both.

#### Verification Script (Unison Test)

```
%%bd_module lsmusic_module.html
%%bd_loop true
%%bd_loop_gap 4
%%bd_reverb_wet 0.35
%%bd_reverb_decay 2.5
%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.2
%%bd_chorus_wet 0.3
%%bd_chorus_depth 0.4
%%bd_tempo 80
%%bd_octave_centre 4

%%bd_voice 1
%%bd_scale pentatonic
%%bd_depth 1
%%bd_angle 90
%%bd_axiom F-F-F-F
%%bd_rules [
F=F+FF-FF-F-F+F+FF-F+F-F-FF+FF+
%%bd_]

%%bd_voice 2
%%bd_scale pentatonic
%%bd_depth 1
%%bd_angle 90
%%bd_axiom GRGRGRG
%%bd_rules [
G=GLGGRGGRGRGLGLGGRGLGRGRGGLGGL
%%bd_]
```

Voice 1 and Voice 2 ABC must be character-for-character identical.

---

### Amendment 11 — Correct Default Script

Replace the default script in `index.html` with the Koch Island two-voice
unison test script from Amendment 10 above. Both voices use pentatonic scale.
Voice 1 uses `F`, `+`, `-` only. Voice 2 uses `G`, `L`, `R` only.
No helper symbols in either voice.

