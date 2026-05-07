# ButterflyDreaming Visual Module — Specification v0.1

## Overview

This document is the source of truth for the first exemplar visual media module
for the ButterflyDreaming platform. It describes two HTML files — a harness and a
visual module — that together demonstrate the BD media module protocol for graphics.
Claude Code should read this document in full before writing any code and refer
back to it throughout.

This spec is a companion to MUSIC_MODULE_SPEC.md. The BD protocol (postMessage,
%%bd_ directives, parseBD parser) is identical. Only the rendering domain differs.

---

## Context

ButterflyDreaming is a dyadic encounter platform documented at butterflydreaming.info.
This visual module renders kolam — Tamil sacred geometry — using L-systems. Kolam
is chosen for its deep resonance with the platform's philosophy: the mandatory
closed-loop constraint mirrors the dyadic encounter that always resolves into a
published child node; the threshold drawing practice mirrors the platform as a
threshold between self and other; the Tamil origin adds a South Indian sacred
geometry tradition alongside the Chinese (Zhuangzi), Western biological
(symbiogenesis) and mathematical (Lorenz) foundations of the platform.

---

## Project Structure

```
butterflydreaming_visual_1/
  VISUAL_MODULE_SPEC.md       ← this file
  index.html                  ← harness (to be created)
  visual_module.html          ← the visual media module (to be created)
```

No sample files are required — all rendering is done in JavaScript on the canvas.

---

## Libraries

Both HTML files load all libraries from CDN — no npm, no build step.

| Library       | Purpose                          | CDN |
|---------------|----------------------------------|-----|
| lindenmayer   | L-System rewriting engine        | https://cdn.jsdelivr.net/npm/lindenmayer/dist/lindenmayer.browser.js |

No other libraries are required. All canvas drawing is vanilla JavaScript.

---

## The BD Directive Syntax

This module uses the BD directive syntax as defined in the BD Node Format
Specification. For reference:

- Single-line directive: terminated by newline
- Multi-line directive: opened by [ on same line, closed by %%bd_] on its own line
- %%bd_module appears once at the top
- Modules interpret directives they understand and silently ignore the rest

---

## The %%bd_ Directives for This Module

### Required
```
%%bd_module visual_module.html
```

### Rendering Directives

| Directive          | Type    | Default    | Description |
|--------------------|---------|------------|-------------|
| %%bd_symmetry      | integer | 8          | N-fold rotational symmetry (1–16) |
| %%bd_depth         | integer | 4          | L-system rewriting iterations |
| %%bd_step          | number  | 40         | Turtle step length in pixels |
| %%bd_angle         | number  | 90         | Turtle turn angle in degrees |
| %%bd_stroke        | color   | #4a9b8e    | Line colour (hex) |
| %%bd_background    | color   | #0a0a0f    | Canvas background colour (hex) |
| %%bd_weight        | number  | 1.5        | Stroke line weight in pixels |

### The Score Directive

The L-system rules are carried in a multi-line %%bd_score block:

```
%%bd_score [
axiom: FBFBFBFB
A: AFBFA
B: AFBFBFBFA
%%bd_]
```

The score block contains:
- One `axiom:` line — the starting string
- One or more production rules in the form `Symbol: replacement`
- Lines beginning with # are comments and are ignored

The symbol alphabet is:
| Symbol | Meaning |
|--------|---------|
| F      | Move forward drawing a curved line segment |
| f      | Move forward without drawing |
| +      | Turn left by %%bd_angle degrees |
| -      | Turn right by %%bd_angle degrees |
| [      | Push current position and angle to stack |
| ]      | Pop position and angle from stack |
| \|     | Turn 180 degrees (reverse direction) |

---

## Default L-System Score

The default score for this module is an 8-fold kolam-inspired pattern that
produces an interlocking closed loop across the canvas.

```
%%bd_module visual_module.html
%%bd_symmetry 8
%%bd_depth 4
%%bd_step 40
%%bd_angle 45
%%bd_stroke #4a9b8e
%%bd_background #0a0a0f
%%bd_weight 1.5
%%bd_score [
axiom: F+F+F+F+F+F+F+F
F: F-F+F+F-F
%%bd_]
```

The angle of 45 degrees and the eightfold rotational symmetry create a more
authentic kolam-like interlocking pattern than a simple 90-degree grid grammar.

---

## The postMessage Protocol

Identical to the music module. Reproduced here for completeness.

### Harness → Module
```javascript
// Load and render a node
{ type: "BD_INIT", payload: { text: "...full node text..." } }

// Stop rendering (clear canvas)
{ type: "BD_STOP" }
```

### Module → Harness
```javascript
// Module loaded and canvas ready
{ type: "BD_READY" }

// User edited text and pressed Send Back
{ type: "BD_UPDATE", payload: { text: "...updated node text..." } }

// Rendering error
{ type: "BD_ERROR", payload: { message: "description" } }
```

During development use origin "*". Note in code that production should
use "https://butterflydreaming.info".

---

## File 1: index.html

### Purpose
Simulates the main ButterflyDreaming platform for development and testing.
Not part of the final platform.

### Explanatory Header
Add an explanatory section at the top of the page, in a calm understated
style, smaller than the main UI elements, off-white at reduced opacity (0.6),
separated from the controls by a thin horizontal rule. Text:

---

SIMPLE EXAMPLE OF TEXT TO MEDIA — ButterflyDreaming Platform

This page demonstrates how a text node from the ButterflyDreaming graph can
drive a visual media module. The L-system rules and %%bd_ directives shown
below would be found in a graph node, discovered and edited collaboratively
by two anonymous users during a dyadic encounter.

The %%bd_ directives are a shared platform language — each directive is
available to all media modules, which interpret them in their own way or
ignore them silently. A directive that controls reverb in the music module
might influence colour or motion in a visual module.

The kolam tradition of Tamil Nadu draws closed loops around a grid of dots
at the threshold of the home at dawn. This module renders kolam using
L-system rewriting — the same mathematical framework as plant growth and
fractal geometry. In ButterflyDreaming, the mandatory closed-loop constraint
of kolam mirrors the dyadic encounter: every path must return to its origin.

---

### Layout
Two-column layout:

**Left column — Controls:**
- Label: EXTENDED L-SYSTEM NOTATION
- Textarea (id: ls-input) pre-populated with the default score above
- Button: "Send to Player"
- Status area showing last message received from module

**Right column — Player:**
- iframe (id: visual-module) pointing to visual_module.html
- Suggest 600x600px — square to suit the radially symmetric output

### Behaviour
1. On page load the textarea is pre-populated with the default score
2. On page load, after a short delay (500ms) to allow the iframe to initialise,
   send BD_INIT automatically so the kolam renders immediately on opening
3. When "Send to Player" is clicked: read textarea, post BD_INIT to iframe
4. Listen for postMessage events:
   - BD_READY → status: "Module ready"
   - BD_UPDATE → update textarea with payload.text, status: "Received update"
   - BD_ERROR → status: "Error: [message]"

---

## File 2: visual_module.html

### Purpose
The visual media module. Receives L-system notation via postMessage, renders
a kolam using L-system rewriting and turtle graphics on a canvas element with
n-fold rotational symmetry.

### Layout
Minimal layout:
- Status line at top (e.g. "Ready", "Rendering...", "Error")
- Canvas element (id: kolam-canvas) filling most of the available space
  — suggest 560x560px, centred
- "Send Back" button at bottom

### Initialisation Sequence
1. On page load, post BD_READY to parent immediately
2. Listen for postMessage events from parent

### BD_INIT Handling

On receiving BD_INIT:

1. Parse the full node text using parseBD() — extract all %%bd_ directives
   and the %%bd_score block
2. Parse the score block to extract axiom and production rules
3. Apply directives to rendering parameters (with defaults for any missing)
4. Run the L-system rewriting using the lindenmayer library
5. Run the turtle interpreter to produce a path for one sector
6. Apply n-fold symmetry stamping to the canvas
7. Check the kolam closed-loop constraint (Option B — warn only)
8. Update status to "Ready" on success

### BD_STOP Handling
Clear the canvas and update status to "Stopped".

---

## The parseBD() Parser

The BD parser is a two-state machine identical to the one specified in the
BD Node Format Specification. It returns a plain object of directive name/value
pairs. The %%bd_score value is the raw multi-line string between the [ and %%bd_].

```javascript
function parseBD(text) {
  const directives = {}
  const lines = text.split('\n')
  let state = 'text'
  let currentDirective = null
  let currentLines = []

  for (const line of lines) {
    if (state === 'text') {
      if (line.startsWith('%%bd_')) {
        const spaceIdx = line.indexOf(' ')
        if (spaceIdx === -1) continue
        const name = line.slice(5, spaceIdx)
        const value = line.slice(spaceIdx + 1).trim()
        if (value === '[') {
          currentDirective = name
          currentLines = []
          state = 'bracket'
        } else {
          directives[name] = value
        }
      }
    } else if (state === 'bracket') {
      if (line.trim() === '%%bd_]') {
        directives[currentDirective] = currentLines.join('\n')
        state = 'text'
      } else {
        currentLines.push(line)
      }
    }
  }
  return directives
}
```

---

## The Score Parser

Parse the %%bd_score value to extract axiom and productions:

```javascript
function parseScore(scoreText) {
  const result = { axiom: '', productions: {} }
  for (const line of scoreText.split('\n')) {
    const trimmed = line.trim()
    if (!trimmed || trimmed.startsWith('#')) continue
    if (trimmed.startsWith('axiom:')) {
      result.axiom = trimmed.slice(6).trim()
    } else {
      const colonIdx = trimmed.indexOf(':')
      if (colonIdx !== -1) {
        const symbol = trimmed.slice(0, colonIdx).trim()
        const replacement = trimmed.slice(colonIdx + 1).trim()
        result.productions[symbol] = replacement
      }
    }
  }
  return result
}
```

---

## The L-System Rewriting

Use the lindenmayer library to rewrite the axiom:

```javascript
const lsystem = new LSystem({
  axiom: parsed.axiom,
  productions: parsed.productions
})
lsystem.iterate(depth)
const result = lsystem.getString()
```

`depth` is the value of %%bd_depth (default 4). At depth 4 the string may be
several thousand characters — this is normal and renders quickly.

Important: depth 6 and above can produce extremely long strings and may cause
browser performance issues. Cap the maximum depth at 6 in the UI and the parser.

---

## The Turtle Interpreter

### Key Design Decision — Curved Lines for Kolam Aesthetic

Standard L-system turtle graphics use straight line segments. Authentic kolam
uses flowing curved lines looping around dots. To achieve the kolam aesthetic,
the turtle renderer must use quadratic bezier curves rather than straight lines
for F moves.

Implementation: at each F move, instead of drawing a straight line from current
position to next position, draw a quadratic bezier curve with a control point
offset perpendicular to the direction of travel. The offset amount controls
the curviness — suggest a value of step * 0.3 as a starting point.

```javascript
function drawCurvedStep(ctx, x, y, angle, step) {
  const rad = (angle * Math.PI) / 180
  const nx = x + Math.cos(rad) * step
  const ny = y + Math.sin(rad) * step
  // control point offset perpendicular to direction
  const perpRad = rad + Math.PI / 2
  const cpOffset = step * 0.3
  const cpx = (x + nx) / 2 + Math.cos(perpRad) * cpOffset
  const cpy = (y + ny) / 2 + Math.sin(perpRad) * cpOffset
  ctx.quadraticCurveTo(cpx, cpy, nx, ny)
  return { x: nx, y: ny }
}
```

### Turtle State
```javascript
let state = {
  x: 0, y: 0,       // current position
  angle: 0,          // current heading in degrees
  stack: []          // for [ and ] branching
}
```

### Symbol Dispatch
```javascript
for (const symbol of lsystemResult) {
  switch (symbol) {
    case 'F': // move forward drawing curved line
    case 'f': // move forward without drawing
    case '+': // turn left
    case '-': // turn right
    case '[': // push state
    case ']': // pop state
    case '|': // reverse (turn 180)
    // all other symbols: ignore silently
  }
}
```

All symbols not in the above list are silently ignored — this is important
for L-system symbols used only as rewriting placeholders (e.g. A, B, X).

---

## The Symmetry Wrapper

The symmetry wrapper is applied at the renderer level, not in the grammar.
Draw the turtle output once into an offscreen canvas, then rotate and stamp
it n times around the centre:

```javascript
function applySymmetry(ctx, offscreen, symmetry, canvasSize) {
  const cx = canvasSize / 2
  const cy = canvasSize / 2
  const sectorAngle = (2 * Math.PI) / symmetry

  ctx.save()
  ctx.translate(cx, cy)
  for (let i = 0; i < symmetry; i++) {
    ctx.rotate(sectorAngle)
    ctx.drawImage(offscreen, -cx, -cy)
  }
  ctx.restore()
}
```

The turtle starts at the centre of the offscreen canvas (or slightly offset
depending on the pattern). Experiment with starting position — for FBFB-type
kolam grammars, starting at the centre works well.

---

## The Kolam Closed-Loop Constraint (Option B — Warn Only)

After turtle interpretation, check whether the final position is approximately
equal to the starting position (within a tolerance of step / 2):

```javascript
function checkClosure(startX, startY, endX, endY, step) {
  const dist = Math.sqrt((endX - startX) ** 2 + (endY - startY) ** 2)
  return dist < step / 2
}
```

If the path does not close, display a subtle warning in the status area:
"Rendered — note: path does not close (kolam constraint not satisfied)"

Do not prevent rendering — this is a creative tool, not a validator.
An unclosed path may still produce interesting and beautiful results.

---

## The Rendering Sequence

The complete rendering sequence on receiving BD_INIT:

1. Parse directives from node text using parseBD()
2. Parse score block using parseScore()
3. Set rendering parameters with defaults for any missing directives
4. Clear canvas, fill with %%bd_background colour
5. Create offscreen canvas same size as main canvas
6. Run L-system iteration using lindenmayer
7. Run turtle interpreter on offscreen canvas, recording start/end positions
8. Apply symmetry wrapper — stamp offscreen canvas n times onto main canvas
9. Check closure constraint, update status accordingly
10. Post BD_READY to parent if this is the first render, otherwise no message

---

## Visual Design

Consistent with the music module aesthetic:

- Background: %%bd_background (default #0a0a0f — near black)
- Canvas border: a subtle 1px teal border (#4a9b8e at 0.3 opacity)
- Status text: off-white (#e8e8e0) at 0.7 opacity, small, below the canvas
- Send Back button: dark background, teal border and text (#4a9b8e)
- Font: Georgia or system-ui serif
- No gradients, no shadows, no animations
- The canvas itself is the visual focus — keep all UI elements minimal

A brief comment at the top of each file:
"ButterflyDreaming Visual Module — [filename] — BD Protocol v0.1"

---

## Error Handling

- If %%bd_score is missing: post BD_ERROR "No score found in node text"
- If axiom is missing from score: post BD_ERROR "No axiom found in score"
- If L-system produces empty string: post BD_ERROR "L-system produced no output"
- If depth > 6: cap at 6 and display a status note "Depth capped at 6"
- If canvas context unavailable: post BD_ERROR "Canvas not supported"
- Wrap all postMessage handlers in try/catch

---

## Testing Checklist

Before considering this exemplar complete, verify:

- [ ] BD_READY posted on page load
- [ ] BD_INIT received → kolam renders on canvas
- [ ] Default 8-fold kolam renders correctly with curved lines
- [ ] Symmetry stamping produces visually correct n-fold pattern
- [ ] Closure check works — console message for open paths
- [ ] BD_STOP clears the canvas
- [ ] Send Back button posts BD_UPDATE with current textarea text
- [ ] Harness textarea updates when BD_UPDATE received
- [ ] Changing axiom or productions and pressing Send to Player re-renders
- [ ] BD_ERROR displayed correctly for malformed score
- [ ] Depth 6 cap works without crashing the browser
- [ ] Renders correctly in Chrome, Firefox and Safari

---

## What This Exemplar Demonstrates

For third-party module developers, this file shows:

1. How to receive BD_INIT and parse the %%bd_ directives and score
2. How to use the parseBD() and parseScore() functions
3. How to use the lindenmayer library from CDN
4. How to implement a turtle graphics renderer on canvas
5. How to apply n-fold symmetry as a renderer wrapper
6. How to post BD_READY, BD_UPDATE and BD_ERROR
7. The expected visual register of a BD visual module
8. That module type (music vs visual) does not affect the BD protocol —
   the postMessage API is identical

---

## Notes for Claude Code

- Do not use any framework — vanilla HTML, CSS and JavaScript only
- Do not use npm or any build tool — CDN only
- Keep each file self-contained — no shared JS files
- Use ES6+ JavaScript throughout
- The lindenmayer library is loaded from jsDelivr CDN as a browser bundle —
  it exposes a global LSystem constructor
- The turtle interpreter must use quadratic bezier curves for F moves,
  not straight lines — this is essential for the kolam aesthetic
- The symmetry wrapper draws to an offscreen canvas first — do not stamp
  directly onto the main canvas in the loop
- All symbols in the L-system result that are not in the turtle alphabet
  (F f + - [ ] |) must be silently ignored
- The default score must render a recognisable 8-fold kolam pattern
- Cap depth at 6 to protect browser performance
- Add clear comments explaining each section, especially the symmetry wrapper
  and the curved line turtle implementation

---

*ButterflyDreaming — a reflective ecosystem — Visual Module Spec v0.1*

### Amendment 1 — Correction to Kolam Philosophical Note and Title

The introductory text in the Context section states that the dyadic encounter
"always resolves into a published child node". This is not accurate — two users
are not required to agree or produce a child node. The encounter may conclude
without a child node being created. The closed-loop constraint of kolam is
therefore an aspiration or metaphor rather than a literal parallel.

The explanatory header text in index.html contains the same statement and
should be softened accordingly. Replace:

"the mandatory closed-loop constraint of kolam mirrors the dyadic encounter:
every path must return to its origin"

with:

"the closed-loop constraint of kolam — every path returning to its origin —
resonates with the aspiration of the dyadic encounter toward resolution
and return"

The title of index.html should be:

BUTTERFLY DREAMING - SIMPLE EXAMPLE OF TEXT TO MEDIA (VISUAL KOLAM)

Respect the capitalisation exactly as shown above.

### Amendment 2 — Default Grammar Revision

The original default L-system score specified:

```
axiom: FBFBFBFB
A: AFBFA
B: AFBFBFBFA
```

This grammar produces only F, A, and B symbols after rewriting — zero turn symbols (+, -).
As a result, the turtle interpreter walks in a straight line to coordinates exceeding the
canvas bounds and produces no visible rendering.

The default score has been replaced with a confirmed working grammar that produces
a recognisable 8-fold closed-loop pattern:

```
axiom: F+F+F+F+F+F+F+F
F: F-F+F+F-F
```

With parameters: `%%bd_angle 45`, `%%bd_depth 4`, `%%bd_symmetry 8`.

This grammar ensures that the rewritten string contains turn symbols and produces
an interlocking pattern suitable for demonstrating the rendering pipeline.

### Amendment 3 — Angle-Tracked Hue Colouring (Stage 1)

Replace the fixed stroke colour with dynamic hue colouring that tracks the
cumulative turtle heading angle. This gives each direction of travel a distinct
colour, making the rotational structure of the kolam directly visible through
colour as well as geometry.

#### Colouring Rule
At each F move, set the stroke colour using the current cumulative turtle
heading angle:

  hsl(angle mod 360, 35%, 65%)

where angle is the current turtle heading in degrees at the moment the F
move begins. S=35 and L=65 are hardcoded in this stage — they will be
exposed as directives and sliders in Amendment 4.

#### Implementation
In the turtle interpreter, before each F move:

```javascript
const hue = ((state.angle % 360) + 360) % 360
ctx.strokeStyle = `hsl(${hue}, 35%, 65%)`
```

The double modulo ensures the hue is always positive even if the cumulative
angle has gone negative through repeated right turns.

#### Signal Directive
Update %%bd_stroke in the default text in index.html from #4a9b8e to the
keyword "angle" as a signal that angle-tracking is active:

%%bd_stroke angle

The module checks if the value of %%bd_stroke is "angle" and activates
angle-tracked colouring. Any other value is treated as a fixed hex colour
as before. This preserves backward compatibility — a module receiving a
node with a hex stroke colour renders it as a fixed colour.

#### Default Text Update
The default text in index.html is updated to include the stroke signal:

%%bd_module visual_module.html
%%bd_symmetry 8
%%bd_depth 4
%%bd_step 40
%%bd_angle 45
%%bd_stroke angle
%%bd_background #0a0a0f
%%bd_weight 1.5
%%bd_score [
axiom: F+F+F+F+F+F+F+F
F: F-F+F+F-F
%%bd_]

#### Stage 2
Exposing S and L as %%bd_saturation and %%bd_lightness directives with
corresponding sliders is deferred to Amendment 4.

### Amendment 3a — Hardcoded S and L Values for Stage 1

Amendment 3 specifies S=35 and L=65 as the hardcoded values for Stage 1.
These are superseded by this amendment for development purposes.

During Stage 1 use:
  S = 100  (fully saturated — maximum colour visibility for development)
  L = 65   (mid-high lightness — visible against dark background)

L may be adjusted during testing if colours are not sufficiently visible
against the %%bd_background of #0a0a0f. The final values for S and L will
be determined by visual inspection before Amendment 4 exposes them as
directives and sliders.

### Amendment 4 — Sliders for All Rendering Parameters

Add sliders to visual_module.html for all rendering parameters. Sliders
are set from incoming %%bd_ directives on BD_INIT and their current values
are written back into the node text via BD_UPDATE when Send Back is pressed.
This follows the same pattern as the music module.

#### Sliders to Add

**Symmetry**
- Integer steps: 1, 2, 3, 4, 6, 8, 10, 12, 16
- Implemented as a dropdown select rather than a slider
- Default: 8
- Directive: %%bd_symmetry

**Depth**
- Integer slider range 1 to 6 (capped at 6 per spec)
- Default: 4
- Directive: %%bd_depth

**Step**
- Linear slider range 10 to 200
- Default: 40
- Directive: %%bd_step

**Angle**
- Linear slider range 5 to 90 degrees
- Default: 45
- Directive: %%bd_angle

**Line Weight**
- Linear slider range 0.5 to 5
- Default: 1.5
- Directive: %%bd_weight

**Saturation (S)**
- Linear slider range 0 to 100
- Left label: "Grey"  Right label: "Vivid"
- Default: 100
- Directive: %%bd_saturation

**Lightness (L)**
- Linear slider range 0 to 100
- Left label: "Dark"  Right label: "Light"
- Default: 65
- Directive: %%bd_lightness

#### Inbound — index.html to Module
When BD_INIT is received, the module sets all sliders to match the
corresponding %%bd_ directive values. If a directive is absent the
slider stays at its default value.

#### Outbound — Module to index.html
When Send Back is pressed, the module reads all current slider values,
updates or inserts the corresponding %%bd_ directives in the node text,
and posts BD_UPDATE to the parent with the complete updated text.

The %%bd_saturation and %%bd_lightness directives are added to the
default text in index.html:

%%bd_module visual_module.html
%%bd_symmetry 8
%%bd_depth 4
%%bd_step 40
%%bd_angle 45
%%bd_stroke angle
%%bd_saturation 100
%%bd_lightness 65
%%bd_background #0a0a0f
%%bd_weight 1.5
%%bd_score [
axiom: F+F+F+F+F+F+F+F
F: F-F+F+F-F
%%bd_]

#### Re-render on Slider Change
Each slider change should immediately trigger a re-render of the canvas
so the user sees the effect in real time without needing to press
Send to Player.

#### Layout
Add a controls panel below the canvas in visual_module.html. Style
consistently with the music module — dark background, teal accents,
subtle labels. Keep the canvas as the dominant visual element.

### Amendment 5 — Player Height, Slider Labels and Depth Cap

#### Part A — Player Height
Double the vertical height of the visual_module.html player window to
allow all sliders and the canvas to be visible simultaneously without
scrolling.

#### Part B — Slider Labels
Remove all secondary lowercase labels from the sliders. The main
capitalised label above each slider is sufficient. This applies to
labels such as "Grey", "Vivid", "Dark", "Light" and any other
secondary descriptive labels currently shown alongside slider controls.

#### Part C — Depth Cap
Increase the maximum depth cap from 6 to 8. Update the depth slider
range from 1-6 to 1-8. Update all references to the cap value in the
code accordingly.

### Amendment 5a — Correction to Player Height

Amendment 5 Part A is clarified. The canvas must remain square at its
current size. The height increase applies to the overall player window
height only — specifically the containing page or scrollable area of
visual_module.html — to accommodate the sliders below the canvas without
scrolling. Do not change the canvas dimensions.

### Amendment 5b — Canvas Size and Layout Correction

The canvas must be square and fill the full width of the module area.
All slider controls are placed below the canvas in the remaining vertical
space. The canvas size should be calculated from the available width of
the module container, not set to a fixed pixel value.
### Amendment 6 — Dropdown and Slider Display Improvements

#### Part A — Depth Dropdown
Change the depth control from a slider to a dropdown select populated
with integer values 1 to 10. Remove the depth cap of 8 from Amendment 5
Part C — the dropdown naturally limits the range. Place the depth
dropdown on the same line as the symmetry dropdown. Both dropdowns
should be only as wide as necessary to display their content.

#### Part B — Slider Values
Display the current numeric value of each slider in small font
immediately adjacent to the slider. The value should update in real
time as the slider is moved.

### Amendment 7 — Updated Default Grammar

Replace the current default grammar with Test 2 which produces a
closer approximation to authentic kolam with confirmed path closure:

%%bd_module visual_module.html
%%bd_symmetry 8
%%bd_depth 3
%%bd_step 40
%%bd_angle 45
%%bd_stroke angle
%%bd_saturation 100
%%bd_lightness 65
%%bd_background #0a0a0f
%%bd_weight 1.5
%%bd_score [
axiom: F+F+F+F+F+F+F+F
F: F+F-F-F+F+F+F-F
%%bd_]

This supersedes the previous default grammar in index.html.

### Amendment 8 — Depth Cap at 5

Cap the maximum depth at 5 for the current grammar which expands
significantly faster than the previous default. Update the depth
dropdown range from 1-10 to 1-5. This supersedes the depth range
specified in Amendment 6 Part A.

### Amendment 9 — Fine Angle Adjustment Slider (Minutes of Arc)

Add a new slider to visual_module.html for fine angle adjustment in
minutes of arc. This works independently of the existing angle slider
and adds a fractional degree offset to it.

#### New Slider
**Angle Fine (minutes of arc)**
- Linear slider range 0 to 59
- Default: 0
- Label: ANGLE FINE
- Displays current value in small font adjacent to slider
- Directive: %%bd_angle_minutes

#### Effective Angle Calculation
The effective turtle turn angle is calculated as:

effectiveAngle = %%bd_angle + (%%bd_angle_minutes / 60)

This is applied wherever the turtle turn angle is used in the renderer.

#### Behaviour
Follows the same pattern as all other sliders — set from incoming
%%bd_angle_minutes directive on BD_INIT, triggers immediate re-render
on change, included in BD_UPDATE payload when Send Back is pressed.

#### Default Text Update
Add %%bd_angle_minutes to the default text in index.html:

%%bd_angle_minutes 0

### Amendment 10 — Colour Speed Control

Add a colour speed parameter that controls how quickly the hue cycles
through the colour wheel relative to the cumulative turtle angle.

#### Colour Speed Formula
Replace the current hue calculation with:

```javascript
const hue = ((state.angle % (360 * speed)) + (360 * speed)) % (360 * speed) / speed
```

Where speed is the value of %%bd_colour_speed. At speed=1 the behaviour
is identical to the current system — one full colour cycle per 360° of
turning. At speed=4 the hue changes four times more slowly, producing
smoother gradients. At speed=0.25 the hue cycles four times faster.

#### New Slider
**Colour Speed**
- Logarithmic scale — slider position range -2 to 4 (the exponent x)
- Actual speed calculated as: speed = 2^x
- Marked positions:
  - -2 = 0.25 (very fast colour cycling)
  -  0 = 1.0  (current behaviour)
  -  2 = 4.0  (slow, smooth gradients)
  -  4 = 16.0 (very slow, almost single colour)
- Default slider position: 2 (speed = 4.0)
- Display the calculated value next to the slider e.g. "4.0x"
- Label: COLOUR SPEED
- Directive: %%bd_colour_speed

#### Behaviour
Follows the same pattern as all other sliders — set from incoming
%%bd_colour_speed directive on BD_INIT, triggers immediate re-render
on change, included in BD_UPDATE payload when Send Back is pressed.

#### Default Text Update
Add %%bd_colour_speed to the default text in index.html:

%%bd_colour_speed 4

**2D VISUAL MODULE — AMENDMENT 11**

**10.1 — Add Copy Script button**
Add a **Copy** button immediately below or alongside the textarea. When clicked it copies the current textarea content to the clipboard and briefly changes its label to **Copied ✓** for 1.5 seconds to confirm the action.

**10.2 — Add Copy Link button**
Add a **Copy Link** button alongside the Copy button. When clicked it:
- Takes the current textarea content
- Encodes it using Base64 (`btoa(unescape(encodeURIComponent(text)))`)
- Constructs a full URL in the form `https://wrcstewart.github.io/butterflydreaming_visual_1/?script=BASE64STRING`
- Copies that URL to the clipboard
- Briefly changes its label to **Link Copied ✓** for 1.5 seconds to confirm

**10.3 — Load script from URL parameter on startup**
On page load, before the existing autoload code fires, check for a `script` query parameter in the URL. If present:
- Decode it using `decodeURIComponent(atob(scriptParam))`
- Populate the textarea with the decoded content
- The existing autoload code then picks it up and renders it automatically as normal

If no `script` parameter is present, the existing default script loads and renders as normal. No change to existing autoload behaviour.

**Note for future music module implementation:** The same URL parameter pattern will apply to the music module, but auto-send will populate the textarea and open the player panel only — playback will await the user pressing Play, respecting browser autoplay restrictions.

2D VISUAL MODULE — AMENDMENT 12
12.1 — Fix URL parameter loading sequence
The URL parameter loading code added in Amendment 11.3 is not overriding the default script because it is executing after the autoload code. Fix the sequencing so that the URL parameter check runs first, before any default script is applied or any autoload is triggered.
The correct sequence on page load must be:

Check for ?script= parameter in the URL
If found — decode it and populate the textarea with the decoded content
If not found — populate the textarea with the default script
Autoload fires using whatever is now in the textarea

Yes exactly — the parent sends a `BD_REQUEST_UPDATE` message to the player, the player responds with `BD_UPDATE` containing the current script with slider values written back in. Clean and explicit.

Here is Amendment 13:

---

**2D VISUAL MODULE — AMENDMENT 13**

**13.1 — Add Receive from Player button (index.html)**
Add a **Receive from Player** button in index.html, positioned between the Send to Player button and the Copy/Copy Link buttons. When clicked it sends a message to the player iframe:
```javascript
{ type: 'BD_REQUEST_UPDATE' }
```
Then waits for the player to respond with a `BD_UPDATE` message, which updates the textarea as it currently does. No other change to the existing `BD_UPDATE` handler in index.html.

**13.2 — Handle BD_REQUEST_UPDATE in player (visual_module.html)**
In visual_module.html, add a handler for the new `BD_REQUEST_UPDATE` message type. When received, the player should immediately respond by posting to the parent:
```javascript
{ type: 'BD_UPDATE', payload: { text: storedText } }
```
where `storedText` is the current script with all slider directive values written back in, exactly as the existing Send Back button does.

**13.3 — Remove Send Back button (visual_module.html)**
Remove the Send Back button and its click event listener from visual_module.html entirely. The push model is replaced by the pull model via BD_REQUEST_UPDATE. No other changes to visual_module.html.

**Note for future platform development:** The push vs pull decision for the collaborative protocol should be revisited during deep design of the dyadic editing session. Both models have merit in that context and the choice should follow from the consent and agency principles in the ethics framework.

---Also remove colour blue from copy button.

Of course. Here it is:

---

### Amendment 14 — Very Fine Angle Adjustment Slider (Seconds of Arc)

Add a new slider to `visual_module.html` for very fine angle adjustment in seconds of arc. This works independently of the existing angle and angle-fine sliders and adds a fractional offset to the effective angle.

#### New Slider
**Angle Very Fine (seconds of arc)**
- Linear slider range 0 to 59
- Default: 0
- Label: ANGLE VERY FINE
- Displays current value in small font adjacent to slider
- Directive: `%%bd_angle_seconds`
- Positioned immediately below the ANGLE FINE (minutes) slider, grouped visually with the degrees and minutes sliders

#### Effective Angle Calculation
The effective turtle turn angle is now calculated as:

```
effectiveAngle = %%bd_angle + (%%bd_angle_minutes / 60) + (%%bd_angle_seconds / 3600)
```

This replaces the two-term formula introduced in Amendment 9. All three terms contribute to a single effective angle value applied wherever the turtle turn angle is used in the renderer.

#### Behaviour
Follows the same pattern as all other sliders — set from incoming `%%bd_angle_seconds` directive on BD_INIT, triggers immediate re-render on change, included in BD_UPDATE payload when BD_REQUEST_UPDATE is received.

#### Default Text Update
Add `%%bd_angle_seconds` to the default text in `index.html`, positioned immediately after `%%bd_angle_minutes`:

```
%%bd_angle_seconds 0
```

---

### Amendment 15 — Angle Drift Automation

Add a drift control to `visual_module.html` that automatically advances the angle by a fixed number of seconds of arc on a 2-second redraw cycle. The three angle sliders (degrees, minutes, seconds) update visually after each redraw, making the carry propagation visible.

#### New Slider
**Angle Very Fine Speed**
- Linear slider range 0 to 10 (seconds of arc per 2-second tick)
- Integer steps
- Default: 0
- Label: ANGLE DRIFT
- Displays current value in small font adjacent to slider
- Directive: `%%bd_angle_drift`
- Positioned immediately below the ANGLE VERY FINE slider

#### Tick Behaviour
On each 2-second tick:
1. Advance the total angle accumulator by the drift value in seconds of arc
2. Propagate carry through the three angle components:
   - `angle_seconds` advances by drift amount; any excess of 60 carries into `angle_minutes`
   - `angle_minutes` wraps at 60, carrying into `angle_degrees`
   - `angle_degrees` wraps at 360
3. Update all three angle slider positions and displayed values to reflect the new state
4. Trigger a full re-render

#### Interval Management
- When drift is set to 0 the interval is cleared and no redraws are scheduled
- When drift is set to any non-zero value the 2-second interval is started if not already running
- Dragging the drift slider from non-zero back to zero clears the interval immediately

#### Behaviour
Follows the same pattern as all other angle-related sliders — set from incoming `%%bd_angle_drift` directive on BD_INIT, written back into the script on BD_REQUEST_UPDATE alongside the current values of `%%bd_angle`, `%%bd_angle_minutes` and `%%bd_angle_seconds`.

#### Default Text Update
Add `%%bd_angle_drift` to the default text in `index.html`, positioned immediately after `%%bd_angle_seconds`:

```
%%bd_angle_drift 0
```
### Amendment 16 — Angle Slider Thumb Visual Hierarchy

Apply a permanent visual hierarchy to the three angle slider thumbs in `visual_module.html` reflecting how frequently each will be adjusted. Implemented entirely in CSS — no JavaScript involved.

The standard browser thumb size is taken as the baseline. Brightness is applied via the `opacity` property.

#### Thumb Styling

**ANGLE (degrees)**
- Thumb size: unchanged (baseline)
- Thumb opacity: unchanged (baseline)

**ANGLE FINE (minutes) — `%%bd_angle_minutes` slider**
- Thumb size: 50% of baseline
- Thumb opacity: 0.5

**ANGLE VERY FINE (seconds) — `%%bd_angle_seconds` slider**
- Thumb size: 25% of baseline
- Thumb opacity: 0.25

#### Implementation
Target each slider by its existing id using both vendor prefixes:

```css
#angle-minutes-slider::-webkit-slider-thumb { ... }
#angle-minutes-slider::-moz-range-thumb { ... }
#angle-seconds-slider::-webkit-slider-thumb { ... }
#angle-seconds-slider::-moz-range-thumb { ... }
```

No changes to `index.html`. No changes to the BD messaging protocol. No new directives.
### Amendment 17 — Depth-Dependent Drift Interval

Replace the fixed 2-second drift interval with a depth-dependent interval table. The interval is re-evaluated at the end of each tick, so a depth change mid-drift takes effect cleanly at the start of the next tick.

#### Interval Table

| Depth | Interval (seconds) |
|-------|--------------------|
| 1 | 0.1 |
| 2 | 0.1 |
| 3 | 0.1 |
| 4 | 0.2 |
| 5 | 1.0 |

#### Implementation
At the end of each drift tick, before scheduling the next timeout, read the current depth value and look up the interval from the table above. Use `setTimeout` rather than `setInterval` so the interval can vary dynamically.

#### Behaviour
- If depth changes while drift is running the new interval takes effect at the start of the next tick
- If drift is zero no timeout is scheduled regardless of depth
- All other drift behaviour from Amendment 15 is unchanged