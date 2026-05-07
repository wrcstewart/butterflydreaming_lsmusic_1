# ButterflyDreaming Music Module — Specification v0.1

## Overview

This document is the source of truth for the first exemplar media module for the
ButterflyDreaming platform. It describes two HTML files — a harness and a music
module — that together demonstrate the BD media module protocol. Claude Code should
read this document in full before writing any code and refer back to it throughout.

---

## Context

ButterflyDreaming is a dyadic encounter platform where two anonymous users
collaboratively explore and weave symbolic text nodes. The platform is documented
at https://butterflydreaming.info.

Media modules are self-contained HTML files that can be triggered from within the
platform to render a text node as sound, image, or other media. They communicate
with the main platform via the browser postMessage API. The text node content is
the single source of truth — all rendering parameters are encoded within it.

This exemplar uses ABC music notation as the text format and plays it back using
real acoustic recorder samples via the Tone.js Sampler.

---

## Project Structure

The project folder is called `butterflydreaming_music_1` and contains:

```
butterflydreaming_music_1/
  MUSIC_MODULE_SPEC.md        ← this file
  harness.html                ← simulates the main platform (to be created)
  music_module.html           ← the media module (to be created)
  bass-recorder/              ← sample files (already present)
    bass_recorder_F#2.mp3
    bass_recorder_C3.mp3
    bass_recorder_G#3.mp3
    bass_recorder_E4.mp3
```

---

## Sample Files

The bass recorder samples are real acoustic recordings from the Versilian Community
Sample Library (VCSL), licensed CC0 (public domain). They are MP3 files at 192kbps
stereo, converted from the original WAV masters.

The four anchor notes and their Tone.Sampler pitch mappings are:

| Filename                  | Tone.Sampler key |
|---------------------------|-----------------|
| bass_recorder_F#2.mp3     | "F#2"           |
| bass_recorder_C3.mp3      | "C3"            |
| bass_recorder_G#3.mp3     | "G#3"           |
| bass_recorder_E4.mp3      | "E4"            |

Tone.Sampler will automatically pitch-shift between these anchor points to cover
all notes in the ABC score.

---

## Libraries

Both HTML files should load all libraries from CDN — no npm, no build step.

| Library   | Purpose                              | CDN |
|-----------|--------------------------------------|-----|
| Tone.js   | Audio engine and Sampler             | https://cdnjs.cloudflare.com/ajax/libs/tone/14.7.77/Tone.js |
| abcjs     | ABC notation parser and renderer     | https://cdnjs.cloudflare.com/ajax/libs/abcjs/6.2.2/abcjs-basic-min.js |

Note: abcjs is used here only for ABC parsing to extract the note sequence.
Visual score rendering is NOT required in this exemplar.

---

## The postMessage Protocol

This is the communication contract between the harness and the music module.
All messages are plain JavaScript objects posted via window.postMessage.

### Harness → Module

```javascript
// Load and play a new ABC score
{
  type: "BD_INIT",
  payload: {
    text: "X:1\nT:Dream\nM:4/4\nL:1/4\nK:Amin\n|A2BC DEFG|\n"
  }
}

// Stop playback
{
  type: "BD_STOP"
}
```

### Module → Harness

```javascript
// Module has loaded successfully and is ready
{ type: "BD_READY" }

// User has edited the ABC text in the module and pressed Send Back
{
  type: "BD_UPDATE",
  payload: {
    text: "X:1\nT:Dream\nM:4/4\nL:1/4\nK:Amin\n|A2BC DEFG|\n"
  }
}

// An error occurred
{
  type: "BD_ERROR",
  payload: { message: "Failed to parse ABC notation" }
}
```

During development, the origin check should use "*". A comment in the code
should note that this should be tightened to "https://butterflydreaming.info"
in production.

---

## File 1: harness.html

### Purpose
Simulates the main ButterflyDreaming platform for development and testing purposes.
This is NOT part of the final platform — it is a development tool that demonstrates
how the main platform will interact with media modules.

### Layout
A clean, simple single-page layout with two columns:

**Left column — Controls:**
- A label: "ABC Notation"
- A `<textarea>` (id: `abc-input`) pre-populated with a short test ABC score
  (see Default ABC Score below)
- A button: "Send to Player" — sends BD_INIT to the iframe
- A status area showing the last message received from the module

**Right column — Player:**
- An `<iframe>` (id: `music-module`) pointing to `music_module.html`
- Should be large enough to show the module UI comfortably — suggest 600x400px

### Behaviour

1. On page load, the textarea is pre-populated with the default ABC score
2. When "Send to Player" is clicked:
   - Read the current text from the textarea
   - Post a BD_INIT message to the iframe
3. Listen for postMessage events from the iframe:
   - BD_READY → update status area: "Module ready"
   - BD_UPDATE → update the textarea with the new text from payload.text
     and update status area: "Received update from module"
   - BD_ERROR → update status area: "Error: [message]"

### Default ABC Score

```
X:1
T:Dream Fragment
M:4/4
L:1/8
Q:1/4=60
K:Amin
|A2 B2 c2 d2|e4 c4|B2 A2 G2 F2|E8|
|A2 c2 e2 c2|A4 E4|F2 G2 A2 B2|c8|
```

This score is in A minor, at a slow tempo (60bpm), within the range of the bass
recorder samples. It should produce a recognisable melodic phrase.

---

## File 2: music_module.html

### Purpose
The media module itself. A self-contained HTML file that receives ABC notation
via postMessage, parses it, and plays it through the Tone.js Sampler using the
bass recorder samples. This file is the reference implementation for third-party
media module developers.

### Layout
A clean, minimal single-page layout:

- A status line at the top showing current state
  (e.g. "Loading samples...", "Ready", "Playing", "Stopped")
- A Play / Pause button (disabled until samples are loaded)
- A Stop button
- A label: "ABC Notation (editable)"
- A `<textarea>` (id: `abc-display`) showing the current ABC text —
  editable by the user
- A "Send Back" button — posts BD_UPDATE to the parent window with the
  current textarea contents

### Initialisation Sequence

1. On page load, immediately post BD_READY to parent — this tells the harness
   the iframe has loaded (even before samples are ready)
2. Begin loading Tone.Sampler with the four bass recorder samples:
   ```javascript
   const sampler = new Tone.Sampler({
     urls: {
       "F#2": "bass_recorder_F#2.mp3",
       "C3":  "bass_recorder_C3.mp3",
       "G#3": "bass_recorder_G#3.mp3",
       "E4":  "bass_recorder_E4.mp3"
     },
     baseUrl: "./bass-recorder/",
     onload: () => {
       // update status to "Ready"
       // enable Play button
     }
   }).toDestination()
   ```
3. Update status to "Loading samples..."
4. Listen for postMessage events from the parent window

### Message Handling

**BD_INIT received:**
1. Store the ABC text
2. Display it in the textarea
3. Parse the ABC using abcjs to extract the note sequence
4. If parsing fails, post BD_ERROR and update status
5. If parsing succeeds, update status to "Ready — press Play"
6. Do NOT auto-play — wait for the user to press Play

**BD_STOP received:**
1. Stop any current playback
2. Update status to "Stopped"

### ABC Parsing and Playback

Use abcjs to parse the ABC string:

```javascript
const parsed = ABCJS.parseOnly(abcText)
```

Extract the notes from the parsed output. Each note has a pitch and duration.
Map ABC pitch names to Tone.js scientific pitch notation:
- ABC uses letter names with octave markers (, for lower, ' for higher)
- Tone.js uses C4 style notation
- Middle C in ABC is C (no modifier) = C4 in Tone.js

Build a sequence of { note, duration, time } objects and schedule them using
Tone.Transport and sampler.triggerAttackRelease().

Use Tone.Transport for timing so that Play/Pause/Stop work correctly:
- Play → Tone.Transport.start()
- Pause → Tone.Transport.pause()
- Stop → Tone.Transport.stop() and reset position to 0

When the sequence ends, update status to "Finished" and reset transport position.

### ABC Pitch to Tone.js Mapping

ABC middle octave (no modifier) maps to octave 4 in Tone.js:

| ABC | Tone.js |
|-----|---------|
| C   | C4      |
| D   | D4      |
| E   | E4      |
| c   | C5      |
| d   | D5      |
| C,  | C3      |
| D,  | D3      |
| C'' | C6      |

Sharps: ^C = C#4, Flats: _C = Db4

### Duration Mapping

ABC note lengths relative to L (unit note length):
- Default L:1/8 means one unit = eighth note = "8n" in Tone.js
- A2 = two units = quarter note = "4n"
- A4 = four units = half note = "2n"
- A/ = half unit = sixteenth note = "16n"

The tempo is set by the Q field in the ABC header (e.g. Q:1/4=60 means
60 quarter notes per minute). Set Tone.Transport.bpm.value accordingly.

---

## Visual Design

Both files should have a calm, minimal aesthetic consistent with ButterflyDreaming:

- Background: very dark (near black) — suggest #0a0a0f
- Text: off-white — suggest #e8e8e0
- Accent: a muted teal — suggest #4a9b8e
- Buttons: subtle, not garish — dark background with teal border and text
- Font: a clean serif or system font — suggest Georgia or system-ui
- No gradients, no shadows, no animation beyond functional feedback
- Textareas: dark background, monospace font for the ABC text

A brief comment at the top of each file should read:
"ButterflyDreaming Media Module — [filename] — BD Protocol v0.1"

---

## Error Handling

- If Tone.Sampler fails to load a sample file, log the error to console and
  post BD_ERROR with a descriptive message
- If ABC parsing produces no notes, post BD_ERROR: "No playable notes found"
- If the browser does not support Web Audio API, display a clear message in
  the status area: "Web Audio not supported in this browser"
- All postMessage handlers should be wrapped in try/catch

---

## Testing Checklist

Before considering this exemplar complete, verify:

- [ ] Samples load without errors in Chrome, Firefox and Safari
- [ ] BD_INIT received → ABC displayed in module textarea
- [ ] Play button plays the score using the bass recorder sound
- [ ] Pause button pauses mid-score and resumes from same position
- [ ] Stop button stops and resets to beginning
- [ ] Editing the module textarea and pressing Send Back updates the harness textarea
- [ ] Editing the harness textarea and pressing Send to Player replays with new score
- [ ] BD_ERROR displayed correctly in harness status for malformed ABC
- [ ] No console errors on page load

---

## What This Exemplar Demonstrates

For third-party module developers, this file shows:

1. How to receive BD_INIT and extract text content
2. How to post BD_READY on load
3. How to post BD_UPDATE when the user modifies content
4. How to post BD_ERROR on failure
5. How to use the bass recorder samples with Tone.Sampler
6. The expected visual register of a BD media module

A third-party developer building a different module (graphics, XR, different
instrument) need only follow the postMessage protocol. Everything else is their
creative freedom.

---

## Notes for Claude Code

- Do not use any framework (no React, no Vue) — vanilla HTML, CSS and JavaScript only
- Do not use npm or any build tool — CDN only
- Keep each file self-contained — no shared JS files between harness and module
- Use ES6+ JavaScript (const, let, arrow functions, async/await) throughout
- Add clear comments explaining each section, especially the postMessage handlers
- The bass-recorder folder is at the same level as the HTML files — use relative paths
- Test ABC score must be within the range F#2 to E4 (the range of the samples)
- Do not auto-play on BD_INIT — always wait for user to press Play

---
## Amendments to Spec v0.1

### Amendment 1 — Key Signature Handling
The ABC parser must honour the key signature declared in the K: field of the
ABC header. When using abcjs parsed output, use the note objects returned by
the parser directly — do not re-parse raw letter names. The abcjs parser
applies key signature accidentals to each note automatically in its output.
This ensures that for example K:D correctly makes all F, C and G notes sharp
without them needing explicit ^ markers in the score.

### Amendment 2 — Play/Pause/Stop Controls
Replace the separate Play and Pause buttons described earlier with:
- A single **Play/Pause toggle button** whose label changes depending on state:
  - Shows "Play" when stopped or paused
  - Shows "Pause" when playing
- A separate **Stop button** that stops playback and returns position to the
  beginning of the score
This is the standard audio player pattern and what users will expect.
*ButterflyDreaming — a reflective ecosystem — Media Module Spec v0.1*
## Amendments to Spec v0.2

### Amendment 3 — Sample Filename Correction
The `#` character in filenames causes URL parsing errors in browsers as it is
interpreted as a fragment identifier. The sharp symbol is therefore represented
as `s` in all sample filenames. The correct filenames are:

| Filename                  | Tone.Sampler key |
|---------------------------|-----------------|
| bass_recorder_Fs2.mp3     | "F#2"           |
| bass_recorder_C3.mp3      | "C3"            |
| bass_recorder_Gs3.mp3     | "G#3"           |
| bass_recorder_E4.mp3      | "E4"            |

Note: The Tone.Sampler url mapping still uses the correct musical notation
("F#2", "G#3") as keys — only the actual filenames use `s` for sharp.
This convention should be followed for all future sample filenames.

### Amendment 4 — Pool-Based Lazy iframe Loading
Iframes are not pre-loaded at startup. Instead the harness creates an iframe
the first time a particular module type is triggered. Once created the iframe
remains in the DOM and is shown/hidden as needed — it is never destroyed or
reloaded. This means samples load once per session per module type.

### Amendment 5 — Cache Assumption
This module assumes browser caching is active in production. During development
the cache may be disabled in DevTools — this is fine but means samples will
reload on every refresh. Do not design modules to work around cache being
disabled. Always use Tone.loaded() to gate playback:

```javascript
await Tone.loaded()
// safe to enable Play button
```

### Amendment 6 — Text Display
The ABC text textarea is removed from music_module.html. Text lives only in
the harness. The module receives text via BD_INIT, plays it, and only sends
text back to the harness via BD_UPDATE when the user explicitly requests it.

### Amendment 7 — Play/Pause/Stop Controls
Confirmed from Amendment 2: a single Play/Pause toggle button plus a separate
Stop button that returns to the beginning of the score.

### Amendment 8 — Rename harness.html to index.html
The harness file should be named `index.html` rather than `harness.html`.
This is the standard convention for the root HTML file of a web project and
means it loads automatically when the project folder or GitHub Pages URL is
opened in a browser without specifying a filename. The iframe src reference
in index.html still points to `music_module.html` unchanged.

### Amendment 9 — Reverb Controls and %%bd_ Parameter Round-Trip

#### Reverb Effect
music_module.html adds a Tone.Reverb effect between the Tone.Sampler and the
destination:

```javascript
const reverb = new Tone.Reverb()
sampler.connect(reverb)
reverb.toDestination()
```

#### Reverb Controls
Two sliders are added to music_module.html:

**Wet/Dry slider:**
- Linear range 0 to 1
- Left label: "Dry"  Right label: "Wet"
- Default: 0.3
- Maps directly to reverb.wet.value

**Decay slider:**
- Slider position range: -1 to 2 (the exponent x)
- Actual decay value calculated as: decay = 10^x seconds
- Default slider position: 0 (= 1 second decay)
- Maps to reverb.decay
- Display the calculated value next to the slider e.g. "1.0s"
- Marked positions: -1 = 0.1s, 0 = 1s, 1 = 10s, 2 = 100s (experimental)

#### %%bd_ Parameter Encoding
Reverb parameters are encoded in the ABC text using the %%bd_ namespace prefix
to avoid collision with standard abcjs directives. These lines appear as plain
text in the ABC header, after the standard fields and before the first bar line.
Example:

%%bd_reverb_wet 0.35
%%bd_reverb_decay 1.8

The value stored for %%bd_reverb_decay is the actual decay time in seconds,
not the slider position. The module converts between seconds and slider position
internally using x = log10(decay).

#### Outbound — Module to index.html
When the user presses Send Back, the module:
1. Reads current slider values
2. Calculates actual parameter values
3. Updates or inserts %%bd_ lines in the ABC text
4. Sends the complete updated ABC text via BD_UPDATE

If %%bd_ lines already exist in the text they are replaced in place.
If they do not exist they are inserted after the last standard ABC header
field and before the first bar line.

#### Inbound — index.html to Module
When BD_INIT is received, the module:
1. Parses the ABC text for any %%bd_ fields
2. If %%bd_reverb_wet found — set wet/dry slider to that value
3. If %%bd_reverb_decay found — convert seconds to slider position
   using x = log10(decay), set decay slider to x
4. If no %%bd_ fields found — leave sliders at current defaults
5. Apply the parameter values to the Tone.Reverb instance immediately

#### Default Values
If no %%bd_ fields are present in the incoming ABC text:
- Wet/Dry: 0.3
- Decay: 1.0s (slider position 0)

#### Developer Note on Tone.Reverb Initialisation
Tone.Reverb generates its impulse response asynchronously. Always await it
before starting playback:

```javascript
await reverb.ready
```

This should be awaited alongside Tone.loaded() during initialisation.

### Amendment 10 — Vibrato Controls and %%bd_ Parameter Extension

#### Vibrato Effect
music_module.html adds a Tone.Vibrato effect in the signal chain after the
Tone.Sampler and before Tone.Reverb:

```javascript
const vibrato = new Tone.Vibrato({
  frequency: 4,
  depth: 0,
  type: "sine"
})
sampler.connect(vibrato)
vibrato.connect(reverb)
reverb.toDestination()
```

The oscillation type is fixed as "sine" and not exposed as a user control.
Depth defaults to 0 — vibrato is off unless the user activates it.

#### Vibrato Controls
Two sliders are added to music_module.html alongside the reverb controls:

**Frequency slider:**
- Logarithmic scale — slider position range -0.3 to 1.3 (the exponent x)
- Actual frequency calculated as: frequency = 10^x Hz
- Marked positions:
  - -0.3 = 0.5 Hz (very slow)
  -  0   = 1.0 Hz
  -  0.7 = 5.0 Hz (classical vibrato)
  -  1.3 = 20 Hz (experimental)
- Default slider position: 0.7 (= 5Hz, classical vibrato rate)
- Display the calculated value next to the slider e.g. "5.0 Hz"
- Maps to vibrato.frequency.value

**Depth slider:**
- Linear range 0 to 1
- Left label: "None"  Right label: "Deep"
- Default: 0 (vibrato off)
- Maps to vibrato.depth.value

#### %%bd_ Parameter Encoding Extension
Two new %%bd_ fields are added to the ABC header parameter set:

%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.0

The value stored for %%bd_vibrato_frequency is the actual frequency in Hz,
not the slider position. The module converts between Hz and slider position
internally using x = log10(frequency).

#### Outbound — Module to index.html
The Send Back action now includes all four %%bd_ parameters:

%%bd_reverb_wet 0.35
%%bd_reverb_decay 1.8
%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.0

All four are written or updated in place in the ABC header on every Send Back.

#### Inbound — index.html to Module
When BD_INIT is received, the module additionally:
1. If %%bd_vibrato_frequency found — convert Hz to slider position
   using x = log10(frequency), set frequency slider to x
2. If %%bd_vibrato_depth found — set depth slider to that value
3. If no %%bd_vibrato fields found — leave sliders at current defaults
4. Apply values to the Tone.Vibrato instance immediately

#### Default Values
If no %%bd_vibrato fields are present in the incoming ABC text:
- Frequency: 5.0 Hz (slider position 0.7)
- Depth: 0.0 (vibrato off)

### Amendment 11 — Corrected Default Values

If no %%bd_ fields are present in the incoming ABC text, all effects default
to their neutral/off state:

- %%bd_reverb_wet: 0.0 (fully dry, no reverb)
- %%bd_reverb_decay: 1.0s (neutral, irrelevant when wet is 0)
- %%bd_vibrato_depth: 0.0 (vibrato off)
- %%bd_vibrato_frequency: 5.0 Hz (neutral, irrelevant when depth is 0)

This supersedes the default values stated in Amendments 9 and 10.

### Amendment 12 — Recommended Initial Values in Harness and Chorus Effect

#### Part A — Recommended Initial Values in index.html
The default ABC text in index.html is updated to include all %%bd_ fields
with musically considered starting values. The module sliders must open in
sync with these values via the BD_INIT round-trip on page load.

The default ABC text in index.html should be:

X:1
T:Dream Fragment
M:4/4
L:1/8
Q:1/4=60
K:Amin
%%bd_reverb_wet 0.35
%%bd_reverb_decay 2.5
%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.2
%%bd_chorus_wet 0.3
%%bd_chorus_depth 0.4
|A2 B2 c2 d2|e4 c4|B2 A2 G2 F2|E8|
|A2 c2 e2 c2|A4 E4|F2 G2 A2 B2|c8|

This supersedes the default ABC text specified in the original spec.

#### Part B — Chorus Effect
music_module.html adds a Tone.Chorus effect in the signal chain between
the Tone.Sampler and Tone.Vibrato:

```javascript
const chorus = new Tone.Chorus({
  frequency: 1.5,
  delayTime: 3.5,
  depth: 0.4,
  wet: 0.3
})
chorus.start()
```

The signal chain is:
Sampler → Chorus → Vibrato → Reverb → Destination

Note: Tone.Chorus requires chorus.start() to be called to begin its
internal LFO. Frequency and delayTime are fixed and not exposed as
controls — only wet and depth are user-controllable.

#### Chorus Controls
Two sliders are added to music_module.html alongside the existing controls:

**Depth slider:**
- Linear range 0 to 1
- Left label: "Thin"  Right label: "Rich"
- Default: 0.4
- Maps to chorus.depth

**Wet slider:**
- Linear range 0 to 1
- Left label: "Dry"  Right label: "Wet"
- Default: 0.3
- Maps to chorus.wet.value

#### %%bd_ Parameter Encoding Extension
Two new %%bd_ fields are added:

%%bd_chorus_wet 0.3
%%bd_chorus_depth 0.4

#### Inbound — index.html to Module
When BD_INIT is received, the module additionally:
1. If %%bd_chorus_wet found — set chorus wet slider to that value
2. If %%bd_chorus_depth found — set chorus depth slider to that value
3. If no %%bd_chorus fields found — use defaults above
4. Apply values to the Tone.Chorus instance immediately

#### Outbound — Module to index.html
The Send Back action now includes all six %%bd_ parameters:

%%bd_reverb_wet 0.35
%%bd_reverb_decay 2.5
%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.2
%%bd_chorus_wet 0.3
%%bd_chorus_depth 0.4

#### Updated Default Values
If no %%bd_ fields are present in the incoming ABC text, all effects
default to the recommended initial values listed in Part A above.
This supersedes Amendment 11.

### Amendment 13 — Window Depth and Title Changes

#### Part A — Window Depth
Increase the height of all windows and panels in both index.html and
music_module.html by 20% to eliminate the need to scroll during normal
use. This applies to the iframe in index.html and all panel elements
in music_module.html.

#### Part B — Title Changes
The following title/heading text changes apply:

In index.html:
- Any occurrence of "MODULE HARNESS" or "Harness" in visible headings
  or titles is replaced with "SIMPLE EXAMPLE OF TEXT TO MEDIA"

In music_module.html:
- Any occurrence of "MODULE" or "Music Module" in visible headings
  or titles is replaced with "SIMPLE EXAMPLE OF TEXT TO MEDIA"

Internal code identifiers, variable names, comments and the postMessage
protocol message types (BD_INIT, BD_READY etc.) are not affected by
this change — only visible UI text.

### Amendment 14 — Textarea Height Correction

Amendment 13 Part A is clarified: the height increase of 20% applies
specifically to the ABC text textarea in index.html only. The player
iframe and all panels in music_module.html are already adequately sized
and should not be changed.

### Amendment 15 — Textarea Height Final Correction

History of textarea height changes:
- Original spec: textarea height unspecified, defaulted to browser default
- Amendment 13: requested 20% height increase — cc did not apply it to textarea
- Amendment 14: clarified 20% increase applies to textarea only — cc applied
  it incorrectly, reducing iframe height instead
- Corrective prompt: restored iframe height, applied partial textarea increase
  — result was still approximately 10% short of target

Final instruction: increase the ABC textarea height in index.html by a further
10% from its current value. This is the final correction and should result in
the textarea being fully visible without scrolling during normal use.

The target state after this amendment is:
- ABC textarea in index.html: original height + 30% total
- Player iframe in index.html: unchanged from its correct size
- All panels in music_module.html: unchanged

### Amendment 16 — Module Type Declaration

A %%bd_ field is added to the ABC header to declare which media module
file should be used to render this text. This allows the harness to
identify and load the correct module from the pool.

The field is:

%%bd_module music_module.html

This line should appear as the first %%bd_ field in the ABC header,
before all other %%bd_ parameters. The updated default ABC text in
index.html is therefore:

X:1
T:Dream Fragment
M:4/4
L:1/8
Q:1/4=60
K:Amin
%%bd_module music_module.html
%%bd_reverb_wet 0.35
%%bd_reverb_decay 2.5
%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.2
%%bd_chorus_wet 0.3
%%bd_chorus_depth 0.4
|A2 B2 c2 d2|e4 c4|B2 A2 G2 F2|E8|
|A2 c2 e2 c2|A4 E4|F2 G2 A2 B2|c8|

The harness reads %%bd_module on receiving any ABC text and uses the
value to identify which iframe to show from the module pool. If no
%%bd_module field is present the harness should default to
music_module.html.

### Amendment 17 — Module Name Display in music_module.html

Add a small text label alongside the player controls in music_module.html
displaying the module filename, drawn from the %%bd_module field:

%%bd_module music_module.html

This should appear as plain text in a subtle style consistent with the
existing UI — not a heading, just a small identifier. For example:

Module: music_module.html

If no %%bd_module field is present in the received ABC text, display
the actual filename as a fallback:

Module: music_module.html

### Amendment 18 — %%bd_ Namespace Convention and Philosophy

The %%bd_ namespace is global across all modules. No module-specific
sub-namespace is used. Directive names are chosen to be self-evidently
descriptive — %%bd_reverb_wet could only ever mean one thing across
the platform. Modules silently ignore any directive they do not implement.

Some directives are intentionally shared across module types. A directive
such as %%bd_intensity may influence the loudness of a music module and
the brightness of a visual module — the same word, different expression.
This ambiguity is a feature consistent with the platform's philosophy of
emergent meaning through encounter.

A published registry of known directives and which modules implement them
will be maintained in the project repository, growing organically as new
modules are contributed.

### Amendment 19 — Explanatory Header Text for index.html

Add an explanatory section at the top of index.html above all controls,
in a calm understated style consistent with the existing UI, smaller than
the main UI elements, in off-white at reduced opacity (suggest 0.6),
separated from the controls below by a thin horizontal rule.

The text should read:

SIMPLE EXAMPLE OF TEXT TO MEDIA — ButterflyDreaming Platform

This page demonstrates how a text node from the ButterflyDreaming graph
can drive a media module. In the live platform the ABC notation and
%%bd_ directives shown below would be found in a graph node, discovered
and edited collaboratively by two anonymous users during a dyadic encounter.

The %%bd_ directives are a shared platform language — each directive is
available to all media modules, which are free to interpret them in their
own way or ignore them silently. A directive that controls reverb in a
music module might influence colour or motion in a visual module.

In ButterflyDreaming the creative process is always collaborative. New
nodes are produced by merge-editing of ancestor nodes, so the set of
directives a user encounters grows gradually through inheritance rather
than being invented from scratch. This protects users from being
overwhelmed by unfamiliar parameters — each new directive arrives with
the context of where it came from.

### Amendment 20 — Revised BD Directive Syntax

This amendment supersedes all previous syntax descriptions including all
semicolon terminators mentioned in Amendments 9 through 17.
Semicolons are not used in the BD directive syntax.

There are exactly two termination rules:

1. A single-line directive is terminated by a newline.
   No special character is needed.

2. A multi-line directive is opened by [ on the same line as the
   directive name and terminated by %%bd_] on its own line.
   The closing marker is namespaced to the platform prefix and cannot
   appear naturally in ABC notation, SVG, JSON, prose, or any other
   content format.

Single-line directives:
%%bd_module music_module.html
%%bd_reverb_wet 0.35
%%bd_reverb_decay 2.5
%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.2
%%bd_chorus_wet 0.3
%%bd_chorus_depth 0.4

Multi-line directive:
%%bd_score [
X:1
T:Dream Fragment
M:4/4
L:1/8
Q:1/4=60
K:Amin
|A2 B2 c2 d2|e4 c4|
|A2 c2 e2 c2|A4 E4|
%%bd_]

The parser is a two-state machine:
- text state: scans for %%bd_ directives, extracts single-line values,
  switches to bracket state when [ is found after a directive name
- bracket state: collects all lines into the directive value until
  %%bd_] appears alone on a line, then returns to text state

### Amendment 21 — Loop Directives

Two new directives control playback looping:

%%bd_loop true
%%bd_loop_gap 6

%%bd_loop controls whether the score repeats after completion.
Default: true

%%bd_loop_gap sets the silence in seconds between the end of the score
and the restart of the sequence. This allows the reverb tail of the
final note to decay naturally before the sequence begins again.
Default: 6

These directives should be added to the default text in index.html.

### Amendment 22 — Updated Default Text in index.html

The complete default ABC text in index.html is:

%%bd_module music_module.html
%%bd_reverb_wet 0.35
%%bd_reverb_decay 2.5
%%bd_vibrato_frequency 5.0
%%bd_vibrato_depth 0.2
%%bd_chorus_wet 0.3
%%bd_chorus_depth 0.4
%%bd_loop true
%%bd_loop_gap 6
%%bd_score [
X:1
T:Dream Fragment
M:4/4
L:1/8
Q:1/4=60
K:Amin
|A2 B2 c2 d2|e4 c4|
|A2 c2 e2 c2|A4 E4|
%%bd_]

This supersedes all previous default text specifications.

### Amendment 23 — Multi-block Sequences (Future Extension)

The directive syntax is designed to support sequential multi-block
execution in a future version. A node may contain multiple %%bd_score
blocks, each preceded by directives that modify only the parameters
that change. The module would play each block in order, applying
directive changes cumulatively between blocks. The %%bd_module directive
appears only once at the top of the node.

This capability is NOT implemented in the current version. A single
%%bd_score block per node is the current limit. Multi-block sequencing
is reserved for a future release once the collaborative editing system
is established and the timing and state-tracking requirements are
better understood.

### Amendment 24 — ABC Notation Box Height and Title

Increase the height of the ABC notation textarea in index.html by 3 lines
from its current value. This is in addition to previous height adjustments
and should eliminate the need to scroll with the current default text.

Change the label above the textarea from:

ABC NOTATION

to:

EXTENDED ABC NOTATION

### Amendment 25 — ABC Chord Notation Support

The current extractNotes function takes only pitches[0] from each element,
silently discarding all other pitches in a chord. This amendment implements
full chord support.

#### Fix to extractNotes
Replace the single pitch extraction with iteration over all pitches in
elem.pitches, pushing one event per pitch at the same time offset:

Instead of:
  const p = elem.pitches[0];
  notes.push({ tonePitch: abcPitchToTone(p.pitch, p.accidental),
               durationWhole: elem.duration });

Use:
  elem.pitches.forEach(p => {
    notes.push({ tonePitch: abcPitchToTone(p.pitch, p.accidental),
                 durationWhole: elem.duration });
  });

Tone.Part and triggerAttackRelease handle simultaneous notes at the same
time offset correctly — no further changes to the playback engine are needed.

#### Updated Default ABC Score for Testing
Update the default score in index.html to include two chords for testing.
Replace the existing score lines with:

%%bd_score [
X:1
T:Dream Fragment
M:4/4
L:1/8
Q:1/4=60
K:Amin
|A2 B2 [CEA]4 d2|e4 [EAc]4|
|A2 c2 e2 c2|A4 E4|
%%bd_]

The chord [CEA] on beat 3 of bar 1 and [EAc] on beat 3 of bar 2 provide
clear audible tests of simultaneous note playback.