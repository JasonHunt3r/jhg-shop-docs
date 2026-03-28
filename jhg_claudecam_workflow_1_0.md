# JHG SVG Toolpath Workflow
Jason Hunter Guitars -- Shop Knowledge Doc

*Most CAM software approaches the design-to-toolpath problem the same way: import a model, auto-generate toolpaths, post-process to G-code. For production shops running high volumes of identical parts, this works well.*

*For boutique work -- complex organic curves, tight tolerances, a small machine with a modest controller buffer -- the default approach has real costs.*

*Conventional CAM linearizes curves into dense sequences of short G1 segments. This degrades motion smoothness on lighter controllers and produces large, controller-taxing files. Fixing it typically requires expert knowledge of post-processor settings and toolpath strategy options that most CAM packages bury behind a learning curve as steep as the software's price.*

*The JHG method takes a different approach. Instead of auto-generating toolpaths and hoping for smooth output, it builds the toolpath geometry explicitly as SVG bezier curves. It measures and corrects offset accuracy at the bezier level before any linearization occurs. It verifies the result visually against the design at every stage.*

*Only then does it feed clean curves to a purpose-built Python generator that produces compact G2/G3 arc commands the controller can execute smoothly. The output is a smaller file, smoother machine motion, and a toolpath that has been verified -- not just simulated.*

*This document defines that workflow. It also demonstrates how a set of composable capabilities -- SVG iteration, bezier accuracy measurement, structural path checking, layer management, spatial orientation, and Python generator handoff -- can be assembled into a coherent shop process using Claude as the CAM preview and verification layer. The individual skills are transferable to any similar problem. This document shows one way they work together.*

---

# MODULE 1 -- SVG ITERATION WORKFLOW

---

## Visualization Hierarchy

Use the most efficient method the task permits. Escalate only when necessary. Do not generate a PNG or bitmap without explicit permission -- if a task seems to require one, say so and wait for a yes before proceeding.

### Level 1 -- SVG Artifact (default)

```
Claude edits SVG markup directly → present_files as .svg → renders inline in claude.ai sidebar
```

**Use for:** anything iterative or illustrative -- labels, arrows, zones, callouts, layout adjustments, color, text. Every iteration is a text edit with no execution overhead.

**Switch to Level 2 when:** geometry must be computed mathematically (coordinate transforms, arc fitting, offset paths). Compute the geometry once in Python, save the SVG, then return to Level 1 for all iteration from that point.

### Level 2 -- Hybrid (computed geometry + SVG iteration)

```
Python runs once to compute geometry → saves .svg → Claude iterates on the SVG directly
```

**Use for:** any diagram where the geometry is derived from math. Python runs exactly once to get the coordinates right. All subsequent iteration happens on the SVG directly -- re-run Python only when a value in the PARAMETERS block changes that affects computed coordinates.

**Pipeline gate:** in the bezier toolpath pipeline, the Python generator is built after the corrected bezier layer is verified and confirmed by Jason. All bezier work -- measurement, correction, splitting -- happens in SVG before any generator code is written. Building the generator before the input is verified wastes tokens and produces code that may need to be rewritten.

### Level 3 -- PNG / Raster (requires permission)

```
Python script → cairosvg or equivalent → PNG file → present_files
```

**Use only when:** Jason explicitly asks for a PNG or bitmap, or the deliverable must be a raster file (print output, external document).

**When a task appears to require a PNG:** state that and ask Jason before generating. Wait for a yes. If a PNG is generated, save the intermediate SVG alongside it -- PNG-only output leaves no editable source for future sessions.

---

## SVG Inline Rendering -- Requirements

For an SVG to render inline in the claude.ai sidebar via `present_files`, all of the following must be true:

```
✓  Extension is .svg (not .txt, not .xml)
✓  xmlns="http://www.w3.org/2000/svg" present
✓  viewBox attribute present
✓  No external dependencies (no external fonts, images, or scripts)
✓  Presented with present_files, not printed to stdout
```

If any of these fail, the sidebar renders nothing or shows a download prompt instead.

---

## Viewer Notes

| Viewer | SVG Support |
|--------|-------------|
| claude.ai sidebar | ✓ Renders inline via present_files |
| Adobe Illustrator | ✓ Full support, preferred editing tool -- see Illustrator Round-Trip below |
| Inkscape | ✓ Functional, not preferred |
| Browser (Chrome/Firefox) | ✓ Drag SVG file onto browser tab |
| macOS Preview | ✓ Opens SVG natively |
| Windows Explorer | ✗ No native SVG preview |

---

## Illustrator Round-Trip -- Strip PGF Metadata

When an SVG is saved from Illustrator, it embeds a `<i:aipgf>` block inside `<metadata>` -- a compressed binary copy of the entire document in Illustrator's native PGF format. On reopen, **Illustrator rebuilds from the PGF data and ignores the SVG DOM entirely.** Any element added to the SVG by Claude (or any text editor) after Illustrator's save will be invisible when the file is reopened in Illustrator.

**The rule:** Strip the entire `<metadata>...</metadata>` block from any SVG that will be edited by both Claude and Illustrator. Do this:
- After saving an SVG from Illustrator, before uploading it for Claude to edit
- In Claude, whenever modifying an SVG that contains an `<i:aipgf>` block

This forces Illustrator to parse the raw SVG DOM on next open. All elements are visible regardless of who added them.

**What you lose:** Illustrator-specific features that SVG can't natively represent -- live text effects, gradient meshes, symbol libraries, some layer metadata. For JHG toolpath verification diagrams -- paths, circles, lines, monospace labels -- nothing of value is lost.

**The failure that prompted this rule:** March 2026 -- a `<polygon>` element added by Claude to an Illustrator-exported SVG rendered correctly in browsers and the claude.ai sidebar but was completely invisible in Illustrator. Multiple attempts to fix via element type, attribute format, and document position all failed. Root cause: Illustrator was rebuilding from PGF, not from SVG. Stripping `<metadata>` resolved it immediately.

---

## File Management Rules

- **Use distinct filenames for each version.** Browser and app caching will serve stale files if the filename doesn't change. Increment a version suffix on every reissue -- this applies to SVGs, PNGs, and any output file presented to Jason.
- **Uploaded files are read-only.** Files in `/mnt/user-data/uploads/` cannot be written to. Always copy to the working directory before editing. Verify the write succeeded by checking the file on disk, not in-memory data.

---

# MODULE 2 -- BEZIER TOOLPATH PIPELINE

---

## Overview

This method produces CNC toolpaths as SVG bezier curves at precise offsets from the design geometry. The toolpath is designed and verified as curves in SVG before any linearization or G-code generation. The result is smoother machine motion because the G1/G2/G3 arc fitter receives clean curve input instead of noisy point clouds.

The pipeline runs from session startup through cut path identification, accuracy measurement, iterative correction, and optional path splitting -- then hands off verified bezier curves to the Python generator. Smoothness checks run at import and after each conversion step.

**The SVG toolpath is the design document.** No linearization until G-code generation.

---

## SVG Coordinate Reminders

For spatial orientation vocabulary, see Reference -- Spatial Language.

- **Y-axis is down** in SVG. Larger Y = further down the document. Matters when computing directions for plunge points and offset normals -- what is "down" in SVG coordinates may not be "down" in the object frame if the document is rotated.
- **Scale:** Always read `SVG_SCALE` from the SCALE block comment in the SVG file. If no SCALE block is present, the current working model uses `2.8346 px/mm` -- derived from a 72 DPI export in Adobe Illustrator on a Mac at 100% document scale. Do not assume this value applies to a different file or setup without confirming. If neither a SCALE block nor a confirmed matching setup exists, stop and ask before proceeding with any measurement or coordinate math.
- **Outward direction:** For a CW polygon in SVG space, the left normal `(-dy, dx)/|d|` points outward from the polygon. For the body outline, outward = away from the body = where the bit center goes for an outside profile cut.
- **NC Y-up vs SVG Y-down:** The NC coordinate transform flips Y. Winding that is CW in SVG space becomes CCW in NC space and vice versa. Always state which space you're working in.

---

## Smoothness Check -- Two Modes

The smoothness check runs at defined pipeline stages to surface problematic path geometry. When issues are flagged, pause and get Jason's input before proceeding.

**On failure in either mode:** Report flagged issues with coordinates and indices. Describe what was found and ask. A segment that looks like an artifact may be intentional geometry -- only Jason knows.

---

### Import Mode -- Structural Checks and Curvature Profile

Run on the incoming Illustrator cutpath before Phase 2 begins. No reference curve is available yet -- this scan checks structural integrity and flags known artifact signatures that cause downstream errors.

**Check 1 -- Path closure:** Verify that the path start and end points connect cleanly. The seam is where Illustrator's offset algorithm most commonly leaves flyaway artifacts. Flag any gap or disconnected segment at the seam with coordinates and gap distance.

**Check 2 -- Continuity gaps:** Verify that the endpoint of each segment matches the start point of the next segment within 0.01px. A gap indicates a broken or disconnected path. Flag with segment indices and gap distance.

**Check 3 -- Duplicate points:** Flag any two consecutive points within 0.001px of each other. These cause division-by-zero in normal computation downstream.

**Check 4 -- Self-intersections:** Flag any location where the path crosses itself. Report intersection coordinates.

**Check 5 -- Spikes:** At each interior point, compute the turn angle between incoming and outgoing segment directions. A turn angle greater than 150° on a section that should be smooth is a spike. Before flagging, check whether the curvature rate of neighboring points is similarly high -- if so, it may be intentional sharp geometry (horn tip, waist corner). Flag with point index, coordinates, turn angle, and neighboring curvature context.

**Check 6 -- Extreme curvature / loops:** Flag any bezier segment where the control point displacement exceeds 3x the segment chord length. This is the signature of a looping bezier that will produce an inside-out cut.

**Check 7 -- Relative segment length** *(experimental -- advisory only, do not block):* Flag any segment whose length is less than approximately 5% of the median segment length on that path AND whose endpoints don't form a smooth continuation with their neighbors. Threshold is unvalidated -- report as advisory, not as a hard failure.

**Curvature profile:** After the structural checks, compute and report the curvature along the path as a summary. This is not pass/fail -- it is a baseline picture of the path's character before correction begins. Example: "Curvature is smooth throughout. One high-curvature region near [location] consistent with horn tip geometry." or "Unexpected curvature spike at [location] -- worth reviewing before proceeding." This gives both Claude and Jason an orientation to the path's overall health.

---

### Comparison Mode -- Normal Deviation from Reference Curve

Run after Phase 3 correction, after Phase 4 splitting, and after any path reversal. A reference curve is available at these stages. The reference changes by stage:

- **Corrected bezier vs. design geometry** -- same normal deviation measurement as Phase 2 accuracy check. Reference is the design geometry lookup table.
- **Python toolpath vs. corrected bezier** -- reference is the corrected bezier. Shoot perpendiculars from the corrected bezier outward and measure where the Python toolpath intersects each normal. This gives the actual inward/outward offset at each location in geometrically meaningful terms, independent of point density on either path.

**What smoothness means in this mode:** a smooth toolpath has normal deviation that varies gradually along the curve. An artifact or spike shows as a point where the normal deviation is wildly inconsistent with its immediate neighbors -- large deviation surrounded by small deviation. The surrounding points define what "normal" looks like locally, so the check is relative, not absolute.

**When to run:**
- Import mode: on the cutpath at Phase 1 import
- Comparison mode: after Phase 3 correction, after Phase 4 splitting, after any path reversal for cut direction

---

## Phase 1 -- Session Startup and Cut Path Identification

### Session Startup Protocol

On upload, Claude reads the document and declares:

1. **Scale widget** -- search for `id` containing `scale-widget`. Read `data-origin-x`, `data-origin-y`, and `data-axis-angle` attributes. Confirm scale value against the SCALE block comment. State what was found.
2. **Centerline** -- identify the centerline segment (see Reference -- Spatial Language). Extend it conceptually to establish the object frame orientation. State the derived rotation angle and which direction is "up."
3. **Declared orientation** -- "Scale widget at [origin], axis angle [N]°. Centerline runs at [N]° from SVG vertical. Up in object frame is toward [SVG direction]. Scale confirmed at [value] px/mm." Jason corrects this if wrong before any work proceeds.

If no scale widget is found, state that and proceed with the SCALE block comment value. If neither exists, stop and ask.

### Locating the Cut Path

**Accepted cut path identifiers** (any of the following in the element's `id` or Illustrator layer name):
- `cutpath`, `cut_path`, `cut-path`, `CutPath`, `cut path`
- `toolpath`, `tool_path`, `ToolPath`
- `offset`, `bit_path`, `finish_path`

**If exactly one candidate is found:**
- Run the smoothness check in import mode
- Run the winding check (see below)
- Rename the layer by appending `cutpath 01` to the existing name: `red ring cutpath 01`. If the layer is already named `cutpath` exactly, rename it to `cutpath 01`. The version number confirms the layer has been seen and processed.
- Confirm the layer identity and the feature it belongs to with Jason
- Present the SVG so Jason can see the confirmed starting state
- Proceed to Phase 2

**If multiple candidates are found:**
- List all candidates with their current names and positions in the layer stack
- Ask Jason which one to use before proceeding

**If no cut path is found:**
- Report which paths are present and ask Jason to generate the offset in Illustrator
- **Illustrator settings:**
  - Object → Path → Offset Path
  - Offset distance = target offset in mm (e.g., `BIT_R = 3.175mm`). Convert to Illustrator's unit setting as needed.
  - Joins: Miter -- Miter limit: 20
  - After generating: strip `<metadata>` block before re-uploading (see Module 1 -- Illustrator Round-Trip)
  - Verify the offset curve is a single closed path, not fragmented
- Wait for re-upload before continuing

### Winding Check

After the cut path is identified, verify its winding direction matches the expected cut type. This is an identity check, not a smoothness check -- it belongs here because it depends on knowing what the path is for.

**Expected winding in SVG space:**
- Outside profiles: CW
- Interior pockets: CCW

A reversed winding is reported and confirmed before any action. A path reversed for cut direction is correct; a path reversed by mistake cuts on the wrong side of the geometry. These look identical to Claude -- only Jason can distinguish them. Flag the reversal with the path name, state which winding would be expected for this feature type, and ask Jason to confirm before proceeding.

---

## Phase 2 -- Measure Accuracy

**Method:**
1. Densely sample the design geometry at 10,000--15,000 points. Store as a lookup table.
2. Sample the cut path at 500--1000 evenly-spaced parameter values.
3. For each sample point on the cut path, find the nearest point on the design geometry. The distance between them is the actual offset at that location.
4. Compute deviation = actual offset - target offset. Positive = too far. Negative = too close (will cut into the part).

**Tolerance targets:**
- `<0.1mm` -- production ready
- `0.1--0.2mm` -- acceptable for MDF templates
- `0.2--0.3mm` -- borderline, correct if practical
- `>0.3mm` -- must correct before use

**Reporting:** Identify which bezier segments contain the problem points. Report per-segment worst deviation so corrections can be targeted.

**Accuracy Overlay:**
Generate a colored-dot overlay on the SVG at each sample point:
- Green dot: `<0.1mm` deviation (good)
- Yellow dot: `0.1--0.2mm` (acceptable)
- Orange dot: `0.2--0.3mm` (borderline)
- Red dot: `>0.3mm` (must fix)

Group all dots for this measurement pass in a named group: `<g id="Accuracy Overlay 01">`. If a previous accuracy overlay already exists in the SVG, increment the number: `Accuracy Overlay 02`, etc. Include a legend with point counts and percentages inside the group.

Accuracy overlays are permanent diagnostic layers -- never stripped from the file. When a new accuracy overlay is added, hide the one that is now 3 back. At any time, at most 2 accuracy overlays are visible simultaneously. They can be regenerated at any time by re-running the measurement step.

---

## Phase 3 -- Correct Control Points

Iteratively adjust the bezier control points of problem segments to bring the offset within tolerance. The curve structure is preserved throughout -- no linearization occurs.

Run the smoothness check after correction completes before proceeding.

### How It Works

A cubic bezier has four points: start (P0), control1 (CP1), control2 (CP2), end (P3). The endpoints are shared with adjacent segments and generally shouldn't move. CP1 has maximum influence on the curve shape near t=1/3, and CP2 near t=2/3, following the Bernstein basis functions:

```
B1(t) = 3(1-t)²t      -- peaks at t=1/3, this is CP1's influence
B2(t) = 3(1-t)t²       -- peaks at t=2/3, this is CP2's influence
```

At any sample point where the offset is wrong:
1. Compute the offset error (distance to design geometry minus target distance)
2. Compute the outward direction (from nearest design point toward the cut path point)
3. Weight the correction between CP1 and CP2 using their Bernstein basis values at that t
4. Nudge each CP along the outward direction by the weighted, damped error

### Algorithm

```python
for each iteration (40-50 total):
    for each sample point at t (31 points, skip t < 0.05 and t > 0.95):
        measure offset error at t
        if |error| < threshold (0.08px ~ 0.03mm): skip

        compute outward normal direction
        compute Bernstein weights: w1 = B1(t)/(B1(t)+B2(t)), w2 = B2(t)/(B1(t)+B2(t))
        correction = -error * damping_factor (0.12)

        CP1 += normal * correction * w1
        CP2 += normal * correction * w2
```

### Key Parameters

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Iterations | 40--50 | Convergence cycles. Diminishing returns past 50. |
| Sample points per segment | 31 | Dense enough to catch mid-segment deviations |
| Damping factor | 0.12 | Prevents oscillation. Too high = overshoots. Too low = slow convergence. |
| Skip threshold | 0.08px (~0.03mm) | Don't correct what's already good enough |
| Endpoint exclusion | t < 0.05, t > 0.95 | Don't fight the endpoint anchoring |

### What This Achieves

Starting from Illustrator's offset (typically 91% within 0.1mm, worst 0.5--0.8mm at high-curvature areas like horn tips), this correction loop reliably achieves:
- 100% of points within 0.1mm
- Worst deviation: ~0.06mm
- Average deviation: ~0.02mm

### When It Doesn't Work

- **Segments shorter than ~5px:** Not enough curve to correct. Usually don't need correction anyway.
- **Segments at sharp corners** (splice/transition zones): The curve intentionally departs from the design geometry here. Exclude from correction.
- **Segments with inflection points:** A single cubic can't maintain consistent offset through an inflection. If correction fails to converge, consider splitting the segment at the inflection.

---

## Phase 4 -- Split Paths for Differential Offsets

When different regions of the same outline require different offset distances -- for example, where a neck pocket wall meets a body outline and each needs its own `BIT_R + WALL_OFFSET` combination -- the single continuous cut path must be divided into independent paths, each corrected separately and then rejoined at clean splice points.

This is distinct from Phase 3 correction. Phase 3 adjusts control points on an existing path to improve its accuracy. Phase 4 is a structural operation: it separates one path into two or more independent paths that will each go through their own Phase 2--3 cycle.

**When Phase 4 is needed:** When two adjacent regions of the design geometry have different target offset values and a single continuous Illustrator offset path cannot represent both correctly. The transition between regions will show as a sharp tangent discontinuity on the offset curve -- a corner where the curve direction changes abruptly.

**General procedure:**

1. **Identify transition points** -- locate the sharp tangent discontinuities on the cut path that correspond to where one region ends and another begins. These are not smooth curve corners; they are direction-change artifacts from Illustrator offsetting across a region boundary.

2. **Separate the regions** -- split the single path at each transition point into two independent open paths. Each path now represents one region only.

3. **Correct each path independently** -- run Phase 2--3 on each path with its own target offset value. The correction algorithm operates on each path as if the other does not exist.

4. **Establish splice geometry** -- the two corrected paths now have open endpoints near each transition point. Extend the endpoint tangent of each path until the two extensions intersect. That intersection point is the splice point. Both paths are trimmed or extended to terminate exactly at the splice point.

5. **Verify endpoint coincidence** -- the endpoint of path A and the endpoint of path B at each splice must share exact coordinates. Verify programmatically. Do not trust visual alignment.

6. **Hand-sculpt transition if needed** -- if the angle at the splice is abrupt, Jason may adjust the transition geometry in Illustrator to create a smooth entry or exit for the bit. This is a finishing step, not a correction step -- the splice geometry is structurally sound before this happens.

Run the smoothness check on each separated path after splitting and again after endpoint adjustment.

---

## Integration with G-Code Pipeline

The finalized SVG bezier toolpath feeds into the Python generator as the source geometry. Three things from the full SVG-to-G-Code Methodology (Shop File Standards) are directly relevant to how the generator consumes these curves. Full methodology lives in Shop File Standards.

### Bezier Curves to Arc Commands

**The core rule: let the controller interpolate.** When the generator pre-linearizes curves into hundreds of G1 segments, GRBL's planner hesitates at each junction -- this is the root cause of rough-feeling curves on the TTC450. Feed the arc fitter clean bezier curves and let GRBL generate its own linear steps from G2/G3 commands.

The arc fitter must be constrained -- greedy fitting produces bad results at inflection points:

```python
ARC_FIT_TOL   = 0.1    # mm -- max deviation from fitted arc to accept it
ARC_MAX_CHORD = 15.0   # mm -- max chord length for a single arc
ARC_MIN_R     = 1.0    # mm -- tighter arcs are likely noise
ARC_MAX_R     = 200.0  # mm -- larger arcs are nearly straight, use G1
```

GRBL also requires exact radius match at both endpoints of every G2/G3 arc -- a mismatch of 0.01mm triggers error 33 and halts execution. The emitter must project every arc endpoint onto the fitted circle and track actual machine position through projected endpoints, not input points. *See [G-Code Hygiene doc, G2/G3 Arc Commands section](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_gcode_hygiene_1_7.md).*

### Coordinate Transform

Before the generator can read SVG paths as NC coordinates, the transform must be established and documented in the `.py` PARAMETERS block:

1. Determine `SVG_SCALE` -- from the SCALE block comment, or derive from a known physical dimension
2. Convert SVG coords to mm -- `px / SVG_SCALE` (e.g., `/ 2.8346` for 72 DPI Illustrator exports)
3. Apply NC transform -- translate to bounding-box-center origin, flip Y axis (SVG Y-down → NC Y-up)

The transform must be invertible: any NC coordinate can be recovered to its original SVG coordinate.

### Offset Stacking and Winding

The SVG toolpath is the bit center line at the finish pass position. All design offsets are baked in -- the generator reads it 1:1 and computes only the leave offsets for the multi-pass sequence:

- Rough pass = finish toolpath + `ROUGH_LEAVE` outward
- Penultimate pass = finish toolpath + `PENULT_LEAVE` outward
- Finish and spring passes = finish toolpath (0mm offset)

**Winding governs offset direction.** The offset functions use the right-hand normal: outward on CW paths, inward on CCW paths (NC Y-up space). After any path reversal for cut direction, winding flips and "outward" functions push the wrong way. Always verify with the dot product check after any reversal:

```python
# dot(offset_vec, outward_normal) > 0 = correct; < 0 = negate the offset amount
assert dot(offset_vec, outward_normal) > 0, "offset direction reversed -- negate ROUGH_LEAVE"
```

*See Shop File Standards, Offset Stacking and Offset Direction sections for full detail.*

---

## Layer Stack and Naming

Every JHG toolpath SVG accumulates layers in a defined sequence.

**Layer sequence in order of creation:**
1. **Design geometry** -- Jason's original Illustrator paths. Never touched by Claude or the Python.
2. **Cutpath layer** -- the raw Illustrator offset path, identified and confirmed in Phase 1. Named with `cutpath` suffix (e.g. `red ring 02 cutpath`). The first draft of the toolpath -- it exists before any correction.
3. **Corrected layer(s)** -- Claude's Phase 3 corrected bezier output. Named by replacing `cutpath` with `corrected` in the cutpath layer name (e.g. `red ring 02 corrected`). The verified source for the Python generator.
4. **Python Path layers** -- Python generator output rendered back into the SVG. Named `Python Path 01 (generated)` and subsequent.

**The Python reads by layer name, not by SVG filename.** The generator looks for a layer whose name ends in `corrected` (without a following numeral) -- the most recently confirmed corrected layer. The SVG filename can be versioned or renamed freely without breaking the generator reference.

---

## Corrected Bezier Layer -- Naming and Color

When Claude writes a corrected bezier path to the SVG:

- **Name:** derived from the cutpath layer name, replacing `cutpath` with `corrected` -- e.g. `red ring 02 cutpath` becomes `red ring 02 corrected`
- **If that name already exists:** append a numeral starting at 2 -- `red ring 02 corrected 2`, `red ring 02 corrected 3`, etc.
- **Group id:** same as the display name, e.g. `<g id="red ring 02 corrected">`

**Color sequence for corrected layers:**
- First correction: purple
- Second correction: orange
- Beyond that: Claude picks a spectrally distinct color and states the choice

---

## Python Path Layers -- Naming and Color

When the Python generator writes its output back to the SVG:

- **Name:** `Python Path 01 (generated)`, `Python Path 02 (generated)`, etc. -- sequential, zero-padded
- **Group id:** matches the display name exactly, e.g. `<g id="Python Path 01 (generated)">`

**Color sequence for Python layers:**
- Python Path 01: fuchsia `#ff00ff`
- Python Path 02: lime green `#32cd32`
- Python Path 03: sunset orange `#ff6b35`
- Python Path 04: yellow `#ffe033`
- Beyond that: Claude picks a spectrally distinct color and states the choice

Colors are assigned by slot number, not by run. If Python Path 01 is hidden, the next run is still Python Path 02 in lime green.

---

## Visibility Philosophy

JHG toolpath SVGs follow a rolling visibility scheme: at any time, only the two most recent layers of each type are visible, with older layers hidden but preserved. This applies to corrected layers, Python Path layers, and accuracy overlays independently. The verified corrected layer is always visible and is never hidden by the Python generator.

Nothing is ever deleted -- hidden layers are recoverable at any time by Jason in Illustrator or by Claude on request.

---

## Visibility Rules -- Claude's Responsibility During Correction

The cutpath stays visible throughout so Jason can see the correction against the raw offset.

**Rolling hide rule during correction:** when a new corrected layer is added, hide the layer that is now 3 back. At any time, at most 2 corrected layers are visible simultaneously alongside the cutpath.

| State | Visible | Hidden |
|-------|---------|--------|
| After corrected 1 | cutpath, corrected 1 | -- |
| After corrected 2 | cutpath, corrected 1, corrected 2 | -- |
| After corrected 3 | cutpath, corrected 2, corrected 3 | corrected 1 |
| After corrected 4 | cutpath, corrected 3, corrected 4 | corrected 1, corrected 2 |

---

## Visibility Rules -- Python Handoff

When the Python generator runs for the first time on a given SVG, it fires the handoff hide in addition to writing its overlay. The handoff hide fires once only -- on the first `Python Path 01` write.

**Handoff hide (Python Path 01 only):**
- Hide the `cutpath` layer
- Hide the oldest corrected layer that would otherwise remain visible after the rolling hide rule has been applied

After handoff, the visible state is: verified corrected layer + Python Path 01.

**Example -- if there were 2 corrected layers at handoff:**
- Corrected 1: hidden (oldest of the visible pair)
- Corrected 2: visible (the verified source)
- cutpath: hidden
- Python Path 01: visible

**Example -- if there was only 1 corrected layer at handoff:**
- Corrected 1: visible (the verified source)
- cutpath: hidden
- Python Path 01: visible

---

## Visibility Rules -- Python's Responsibility During Iteration

The Python manages only its own layer visibility -- it writes the new overlay, hides the layer that is now 3 back, and on first run fires the handoff hide. Everything else is off limits.

| State | Visible | Hidden |
|-------|---------|--------|
| Python Path 01 | corrected (verified), P01 | cutpath, older corrected |
| Python Path 02 | corrected (verified), P01, P02 | cutpath, older corrected |
| Python Path 03 | corrected (verified), P02, P03 | cutpath, older corrected, P01 |
| Python Path 04 | corrected (verified), P03, P04 | cutpath, older corrected, P01, P02 |

The verified corrected layer is always visible. It is never hidden by the Python.

---

## What the Python Writes -- Scope

1. Writing the new `Python Path 0N (generated)` layer with its assigned color
2. Hiding the Python layer that is now 3 back (if it exists)
3. On first run only: the handoff hide (cutpath + oldest remaining corrected layer)

**The Python touches nothing else.** Any other visibility state is either already correct from Claude's correction work or is Jason's to manage in Illustrator.

---

## Overlay Limitation -- Shape vs. Traversal

Generator toolpath overlays verify **where** the bit cuts -- the polygon shapes and offset distances are correct. They do **not** verify **how** the bit gets there -- the traversal order, plunge-to-first-corner path, or return pass routing.

Bugs in traversal order (wrong corner sequence, diagonal crossings, return passes starting from the wrong side) are invisible in the overlay SVG because the overlay draws the polygon outline, not the emission sequence.

**When to suspect a traversal bug:** If a test cut shows diagonal witness marks, gouges across a pocket interior, or unexpected straight-line artifacts that don't correspond to any polygon edge -- these are traversal order problems, not shape problems. The overlay will look correct.

**To diagnose:** Inspect the NC file directly. Check the first G1 move after each plunge or pass transition -- its distance should be short (to the nearest corner or along the current side). A long first move (>10mm on a small feature) indicates the command sequence doesn't start where the machine is positioned.

**Example failure:** Panel C EMG ears -- the overlay showed correct rectangles at correct offsets, but the return pass cut 41mm diagonally across the pocket because `find_longside_plunge` on the reversed vertex list picked the wrong side. The overlay couldn't show this because it only drew the rectangle shape.

---

# REFERENCE -- SPATIAL LANGUAGE

*Orientation vocabulary and frame awareness rules for spatial communication between Claude and Jason.*

---

## Vocabulary

- **SVG frame** -- raw SVG coordinate space. X increases right, Y increases down. Used for math, code, and coordinate output.
- **Document window** -- what Jason sees on screen. Upper-left of the Illustrator canvas is top-left, regardless of what's in the document. Jason may naturally refer to positions this way -- "the right corner of the document" means document window, not object frame.
- **Object frame** -- the natural orientation of the thing being worked on. Derived from the centerline and/or widget when the object is rotated relative to the document. "Up" = toward the top of the object along its functional axis.
- **Centerline** -- the functional axis of a document or feature. May be offset from the bounding box center. Read from the document as given -- if no centerline is visible, use the heuristics below to find one. The centerline defines the object frame.
- **Widget** -- the scale reference object (`id` containing `scale-widget`). Has a declared origin point and two perpendicular ruler axes. The 1" box sits inside the 90° sector defined by the axes -- like fair territory in a baseball field, with the origin at home plate. Contains directional labels (TOP, BOT, L, R) oriented to the widget's own axis frame. See [jhg_scale_widget_1_0.md](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_scale_widget_1_0.md) for full spec and generator.
- **Cutpath** -- the confirmed cut path layer in the SVG, identified in Phase 1. Named with `cutpath 01` suffix.

---

## Frame Awareness

Jason may switch between frames naturally mid-conversation without flagging it -- "the right corner of the body" and "the right corner of the document" mean different things and both are valid ways to talk.

Read context and determine which frame Jason is using. When it's clear, proceed. When it's genuinely ambiguous, ask once -- "do you mean right in the document window or right relative to the body?" -- and move on. When stating a position, name the frame if there's any chance of ambiguity.

**Finding the centerline when nothing is labeled:** look in order of confidence:
1. A line passing through or near the center of a feature, running roughly parallel to its long axis
2. A line that multiple features appear to be symmetrically arranged around
3. A long straight line that isn't a cut path, outline, or construction geometry -- ask once to confirm

Multiple centerlines may exist, each associated with a specific object or region -- resolve from context, ask once if ambiguous. If no centerline can be identified, state that and use SVG frame.

---

## Scale Widget -- Session Protocol

On upload of any JHG SVG containing a scale widget:

1. Find the element with `id` containing `scale-widget`
2. Read `data-origin-x`, `data-origin-y`, `data-axis-angle` attributes
3. The widget origin is the corner where the two ruler axes meet -- home plate
4. The directional labels inside the 1" box (TOP, BOT, L, R) are authoritative for object frame orientation -- read them directly
5. Confirm scale against the SCALE block comment
6. Declare all of this at session start before any spatially-dependent work: "Scale widget at [origin], axis angle [N]°. Object frame up is toward [direction]. Scale confirmed at [value] px/mm."

If the widget labels and the derived centerline orientation disagree, flag the discrepancy and ask before proceeding.

*Full widget specification, generator function, and placement conventions: [jhg_scale_widget_1_0.md](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_scale_widget_1_0.md)*

---

## Profile: 17° Body Rotation

The JHG guitar body SVG family is rotated 17° CCW from natural orientation to fit the 18×18" CNC bed. This is the most common case where document window and object frame visibly diverge.

**In these documents:**
- Object frame "up" = toward the neck end, which is toward the upper-right of the document window
- Document window "upper left" ≠ object frame "upper left"
- Widget is placed on the neck centerline with `data-axis-angle="17"`
- Min/max coordinate sorting gives document window corners, not object frame corners -- use the widget and centerline to find object frame positions

When Jason says "top of the body" he means neck end. When he says "top of the document" or "upper left corner" he may mean the document window. Read context -- if a guitar body feature is being discussed, default to object frame. If the document canvas or a panel position is being discussed, default to document window.

**The failure that prompted this profile:** March 2026 -- coordinate-sorted "lower-left" (min X, max Y) on rotated EMG rects was actually the upper-left in object frame because the rectangles are rotated 17°. Three placement attempts were needed before the frame confusion was identified.

---

## Profile: [Next Recurring Setup]

*Add a profile here when a new document family with a consistent rotation or orientation convention is established. Include: rotation angle and reason, widget placement convention, which directions diverge between document window and object frame, and any Jason shorthand that maps to object frame positions.*

---

## Arrow and Annotation Direction

When an arrow points to the **outside face** of a convex curve:
- "Outside" = the convex face = the face pointing *away* from the pocket interior
- Tip position = `curve_extreme + STANDOFF` in the outward direction
- The sign of the standoff offset determines which face you're pointing at -- getting it wrong points to the inside face instead

---

## Quadrant Convention

When a canvas is divided into panels or quadrants: **A=top-left, B=top-right, C=bottom-left, D=bottom-right**

State this explicitly at the start of any session working on a multi-panel layout.

---

## Neck Pocket Specific

Zone color codes, orientation vocabulary (TOP/BOTTOM), and fit shorthand (Z1-TIGHT, Z3-LOOSE, etc.) for the neck pocket fit reference diagram are job-specific and live in [jhg_neck_pocket_details.md](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_neck_pocket_details.md).
