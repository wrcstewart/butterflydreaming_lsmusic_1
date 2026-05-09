
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


Good thinking. Here it is:

---

### Amendment 1 — Correct Duration Mapping to Prusinkiewicz Method

The initial implementation treated every horizontal `F` segment as a note of equal duration. This is incorrect. The Prusinkiewicz method (section 6.3, Figure 6.9) states explicitly that note duration is proportional to segment length, where segment length is the count of consecutive horizontal `F` steps forming a continuous horizontal run.

#### Corrected Segment Classification

During turtle interpretation, do not emit a note per `F` step. Instead, accumulate consecutive horizontal `F` steps into runs. A run ends when any of the following occurs:

- A turn symbol (`+` or `−`) is encountered
- A vertical `F` step occurs (i.e. the heading is vertical at that point)
- The string is exhausted

Each completed horizontal run produces exactly one note.

#### Corrected Duration Mapping

The duration of the note produced by a horizontal run is proportional to the number of `F` steps in that run:

```
note_duration_seconds = run_length × (60 / %%bd_tempo)
```

A run of length 1 produces a quaver-equivalent, length 2 a crotchet-equivalent, and so on. All durations scale with tempo as before.

#### ABC Generation

Express variable note lengths correctly in the generated ABC notation. A run of length `n` at the base `L:1/8` unit produces a note of length `n` in ABC terms (e.g. `C2` for length 2, `C4` for length 4).

#### No Other Changes

Pitch mapping, scale lookup, Y normalisation, and all other behaviour are unchanged.
Good pragmatic approach — fix one thing at a time and get proper visibility of what's being generated.

Here is Amendment 2:

---

### Amendment 2 — ABC Debug Display and Note Duration Fix

#### Part A — ABC Debug Display

Add a read-only text area below the script input box in `index.html` labelled `GENERATED ABC`. This text area:

- Is populated every time the module posts a `BD_UPDATE` message containing a `generatedABC` field
- Is read-only but selectable and scrollable
- Has sufficient height to show approximately 10 lines without scrolling
- Is clearly labelled as debug output

In `lsmusic_module.html`, include the generated ABC string as an additional field `generatedABC` in every `BD_UPDATE` message payload. This field is for debug purposes only and is not written back into the node text.

#### Part B — Note Duration Fix

The ABC generator must implement run-length accumulation for horizontal segments as specified in Amendment 1. This fix must operate at the ABC generation stage, not at the Tone.js scheduling stage.

Specifically:

- Consecutive horizontal `F` steps at the same Y coordinate without an intervening turn are accumulated into a single run of length `n`
- The ABC note for that run is expressed as `Xn` where `X` is the pitch letter and `n` is the run length (e.g. `E2`, `G4`)
- A run of length 1 is expressed without a length suffix (e.g. `E` not `E1`)
- The Tone.js scheduler must read note durations from the ABC, not recalculate them independently

---

Ready for Claude Code.

Yes — the debug display is the priority, without it we're flying blind.

The most likely cause is that `lsmusic_module.html` is not including `generatedABC` in its `BD_UPDATE` payload, or it is only sending `BD_UPDATE` in response to `BD_REQUEST_UPDATE` rather than also sending it after a successful `BD_INIT` parse.

Here is Amendment 3:

---

### Amendment 3 — Fix ABC Debug Display Population

#### Problem
The generated ABC text area in `index.html` is not populating when Send to Player is pressed.

#### Fix

In `lsmusic_module.html`, after a successful `BD_INIT` sequence — once the L-system has been iterated, the turtle has run, and the ABC has been generated — immediately post a `BD_UPDATE` message to the parent containing the `generatedABC` field. Do not wait for a `BD_REQUEST_UPDATE`.

```javascript
window.parent.postMessage({
  type: 'BD_UPDATE',
  text: currentNodeText,
  generatedABC: currentABC
}, '*');
```

In `index.html`, the `message` event listener must handle `BD_UPDATE` messages by writing the `generatedABC` field into the debug text area if that field is present.

#### No Other Changes
All other behaviour is unchanged.

---

Short and targeted — Claude Code should be able to nail this in one pass.
### Amendment 4 — Implement ABC Run-Length Accumulation Correctly

#### Problem
The ABC generator is emitting one quaver per `F` step regardless of run length. Amendments 1 and 2 specified run-length accumulation but it has not been implemented correctly in the ABC generation stage.

#### Definition of a Run
A run is a sequence of consecutive horizontal `F` steps where:
- All steps are horizontal (heading is 0° or 180° at time of step)
- No turn symbol (`+` or `−`) intervenes between steps
- No vertical `F` step intervenes between steps

Two horizontal `F` steps at the same pitch but separated by a turn are **not** a run — they are two separate notes. The pitch match is irrelevant; only the absence of an intervening turn or vertical step defines a run.

#### Required Change — Turtle Interpreter
During turtle interpretation, each horizontal `F` step must be tagged with a `consecutive` boolean:
- `true` if the immediately preceding symbol was also a horizontal `F` step
- `false` if the preceding symbol was a turn, a vertical `F` step, or the start of the string

#### Required Change — ABC Generator
Replace the current per-step note emission with run-length accumulation:

```javascript
let i = 0;
while (i < horizontalSegments.length) {
    let run = 1;
    while (i + run < horizontalSegments.length &&
           horizontalSegments[i + run].consecutive === true) {
        run++;
    }
    // emit note with run length suffix if > 1
    abc += horizontalSegments[i].pitch + (run > 1 ? run : '') + ' ';
    i += run;
}
```

Note that run accumulation is based solely on the `consecutive` flag — **not** on pitch equality. Two consecutive horizontal steps of different pitches do not form a run; the first step has `consecutive === false` on the second step, breaking the run at that point.

#### Verification
After this fix, bars 1 and 2 of the depth 4 Hilbert output should no longer read `EEEEEEEE|EEEEEEEE`. Long horizontal traversals at the same Y level should produce notes with length suffixes such as `E4` or `E8` rather than repeated single quavers. The total number of note symbols in the ABC should be significantly less than the total number of `F` steps in the iterated string.

#### No Other Changes
Pitch mapping, scale lookup, Y normalisation, and all audio rendering are unchanged.

CC's own explanation reveals the actual bug. The fix is resetting runs on **any non-F character** including `X`, `Y` and other non-drawing symbols. But those non-drawing symbols are geometrically invisible — they don't change the turtle heading. So `F X F` where both F steps are horizontal with the same heading **is** a genuine run — the X is transparent.

The rule should be: **only `+` and `−` break a run**. Non-drawing symbols like `X` and `Y` are transparent to run accumulation because they don't change the turtle's heading or position.

For the Hilbert grammar `X=-YF+XFX+FY-`, when iterated the F steps between turns always have X or Y symbols between them — so CC's fix breaks every single run, giving all length-1 notes. That's exactly what we're seeing.

### Amendment 5 — Fix Run-Break Logic

#### Problem
The `buildHorizontalSegments()` function resets the run counter on any non-F character, including non-drawing symbols such as `X` and `Y`. This incorrectly breaks geometrically contiguous horizontal runs wherever the grammar places structural symbols between F steps.

#### Correct Rule
A run is broken **only** by:
- A turn symbol (`+` or `−`)
- A vertical `F` step (heading is vertical at that point)

Non-drawing symbols (`X`, `Y`, and any other symbol that neither moves the turtle nor changes its heading) are **transparent** to run accumulation and must not reset the run counter.

#### Required Change
In `buildHorizontalSegments()` or equivalent, change the reset condition from:

```javascript
// WRONG — resets on any non-F character
if (symbol !== 'F') prevHorizF = false;
```

To:

```javascript
// CORRECT — only turns break a run
if (symbol === '+' || symbol === '-') prevHorizF = false;
```

#### Verification
After this fix, the depth 4 Hilbert output should contain notes with length suffixes. The opening bars should no longer be `EEEEEEEE|EEEEEEEE`.

Exactly right — the pitch mapping is now fixed, giving proper spread. The last remaining issue is the same-pitch merging. Here is Amendment 6:

---

### Amendment 6 — Same-Pitch Note Merging

#### Problem
The ABC output contains consecutive notes at the same pitch written as separate quavers (e.g. `C C`) rather than a single longer note (e.g. `C2`). This happens because the duration is computed before same-pitch merging.

#### Fix
After pitch mapping and before ABC generation, add a post-processing pass over the note array that merges consecutive notes of identical pitch into a single note with combined duration:

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

This pass must run **after** `mapYToPitch()` and **before** `buildABC()`. The ABC generator already handles variable `runLength` correctly via ties across barlines.

#### Expected Result
No two consecutive notes in the ABC output should share the same pitch letter and octave. `C C` becomes `C2`, `G,G,G,G,` becomes `G,4`, and so on.

#### No Other Changes
Pitch mapping, scale lookup, turtle interpreter, and audio rendering are unchanged.

The Chinese quality makes sense — pentatonic scale on a recorder timbre with vibrato is very close to a dizi or xiao. A happy accident that fits the BD aesthetic rather well.

On the Request Update bug — this is the same class of issue that appeared in the visual module. The BD_REQUEST_UPDATE handler is probably reconstructing the node text from the original parsed directives rather than reading the current live values from the slider controls.

### Amendment 7 — Fix BD_REQUEST_UPDATE Slider Values

#### Problem
When `BD_REQUEST_UPDATE` is received, the returned node text does not reflect current slider positions. The handler is reading from stored parsed directive values rather than the live control state.

#### Fix
In the `BD_REQUEST_UPDATE` handler, reconstruct the node text by reading current values directly from every control element at the moment the request is received:

- Depth selector — read `.value`
- Scale selector — read `.value`
- Tempo slider — read `.value`
- Octave centre — read `.value`
- Loop checkbox — read `.checked`
- Loop gap — read `.value`
- Reverb wet, reverb decay — read `.value`
- Vibrato frequency, vibrato depth — read `.value`
- Chorus wet, chorus depth — read `.value`
- Axiom field — read `.value`
- Rules textarea — read `.value`

The generated ABC is **not** included in the returned text. The `generatedABC` field is included in the payload for the debug display as per Amendment 2.

#### No Other Changes
All other behaviour unchanged.

### Amendment 8 — Debug Label, Copy Buttons and URL Parameter Loading

#### 8.1 — Rename Debug Label
Change the label of the generated ABC text area from `GENERATED ABC` to `GENERATED ABC SCORE`.

#### 8.2 — Add Three Copy Buttons
Add three buttons in `index.html` positioned below the script textarea, in this order:

**COPY ABC SCORE**
Copies the current content of the GENERATED ABC SCORE text area to the clipboard. Briefly changes label to `Copied ✓` for 1.5 seconds. Disabled if the text area is empty.

**COPY SCRIPT**
Copies the current content of the script textarea to the clipboard. Briefly changes label to `Copied ✓` for 1.5 seconds.

**COPY LINK**
- Takes the current script textarea content
- Encodes it using `btoa(unescape(encodeURIComponent(text)))`
- Constructs a URL in the form:
  `https://wrcstewart.github.io/butterflydreaming_lsmusic_1/?script=BASE64STRING`
- Copies that URL to the clipboard
- Briefly changes label to `Link Copied ✓` for 1.5 seconds

#### 8.3 — Load Script from URL Parameter on Startup

On page load the correct sequence must be written as a single linear flow. Do not split into separate initialisation blocks.

> ⚠ **Known sequencing pitfall** — also encountered in the 2D visual module (VISUAL_MODULE_SPEC.md Amendment 12). The URL parameter check must execute before any default script is applied or any auto-send is triggered. The entire sequence must be:

```javascript
// CORRECT sequence — do not reorder
const scriptParam = new URLSearchParams(window.location.search).get('script');
if (scriptParam) {
    textarea.value = decodeURIComponent(atob(scriptParam));
} else {
    textarea.value = DEFAULT_SCRIPT;
}
// auto-send fires here using whatever is now in textarea
// does NOT autoplay — populates player only, respects browser autoplay restrictions
sendToPlayer();
```

If no `script` parameter is present the default node text loads as normal. Auto-send populates the player but does **not** trigger playback — the user must press Play, respecting browser autoplay restrictions.

#### No Other Changes
All other behaviour unchanged.

---

### Amendment 9 — Corrections to Copy Link Encoding and URL Parameter Loading

Two bugs prevented the Copy Link / URL parameter round-trip from working.

#### Bug 1 — Encode: base64 `+` characters corrupted by URL parsing

The original encode was `btoa(unescape(encodeURIComponent(textarea.value)))`. The base64 alphabet includes `+`, `/`, and `=`. When this string was placed directly in the URL without further encoding, `URLSearchParams.get()` applied `application/x-www-form-urlencoded` parsing, which treats `+` as a space. This corrupted the base64 before `atob()` could decode it.

**Fix:** Replace the base64 chain with a direct `encodeURIComponent(textarea.value)`. Simpler, no base64, no `+` ambiguity.

#### Bug 2 — Decode: double percent-decoding of `%%` directives

`URLSearchParams.get()` already percent-decodes the query parameter value once. The script directives begin with `%%`, which `encodeURIComponent` encodes as `%25%25` in the URL. `URLSearchParams.get()` decodes `%25%25` back to `%%` correctly. But the code then called `decodeURIComponent()` a second time on that result. `decodeURIComponent("%%bd_...")` throws `URIError` because `%%` is not a valid percent-escape sequence. This error was silently caught and the textarea was reset to the default script.

**Fix:** Use `scriptParam` directly — no `decodeURIComponent` call. `URLSearchParams.get()` has already done the decoding.

#### Final correct implementation

```javascript
// Encode (Copy Link button)
const url = `https://.../?script=${encodeURIComponent(textarea.value)}`

// Decode (page load)
const scriptParam = new URLSearchParams(window.location.search).get('script')
if (scriptParam) {
    textarea.value = scriptParam          // URLSearchParams already decoded it
} else {
    textarea.value = DEFAULT_SCRIPT
}
```

---

## Phase 2 — Two Voices

*Work from this point onwards concerns the addition of a second independent voice to the module.*