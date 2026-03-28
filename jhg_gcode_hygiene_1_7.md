# JHG G-Code Hygiene
Jason Hunter Guitars — TwoTrees TTC450 PRO Standards
*These are the standing rules for every G-code file generated for JHG. Read this before writing or editing any NC file.*

---

## MACHINE CONSTANTS

```
Machine:      TwoTrees TTC450 PRO
Controller:   MKS DLC32 V2.1, ESP32, GRBL-based
Spindle:      500W, ER11 collet
Bed:          457.2 x 457.2 x 80mm (never exceed Z=80mm)
RPM:          18000 (default)
Feed:         1500 mm/min (rough default)
Plunge:       500 mm/min (default)
Safe Z:       12mm
Park Z:       50mm
Bit 1:        1/4" upcut spiral — BIT_R = 3.175mm
Bit 2:        1/8" upcut spiral — BIT_R = 1.5875mm
```

---

## OUTPUT FORMAT RULES

**Confirmed:**
- Semicolons only for comments — no parentheses `( )`
  - Parentheses cause silent SD card execution failure on this machine
  - This is machine-specific, not a general GRBL rule

**Standing hygiene (apply to every file):**
- Pure ASCII only — no Unicode, no special characters, no em-dashes, no curly quotes
- No `%` delimiter — legacy serial artifact, no function in SD card execution
  - Flag and remove if found in any inherited or existing file
- Filenames: alphanumeric, dash, and underscore only
  - No spaces, no `+`, no special characters
  - No periods except the extension
  - Maximum 30 characters
  - Use hundredths integer encoding for offsets: `p40` = +0.40mm, `m40` = -0.40mm

---

## MODAL SETUP BLOCK

Every file must open with this block in this order:

```gcode
G21       ; millimeters
G17       ; XY plane
G90       ; absolute positioning
G94       ; feed per minute
M3 S18000 ; spindle on
G4 P2     ; dwell 2sec - let spindle spool up
G0 Z12    ; safe height
```

---

## GEOMETRY DEFAULTS

- **Origin:** Bounding box center = XY0. Work zero on machine must be set to match.
- **Polygon winding:** CONFIRMED March 2026 from Panel A test cut.
  - Body outline (outside profile on MDF): path runs C2→perimeter→C1 (reversed from SVG direction) for conventional cut. CW-dominant arcs in NC space.
  - EMG pickup pockets (interior pockets on MDF): CCW winding in NC space for conventional cut. Reverse the polygon vertices after offset computation.
  - The `offset_polygon_outward()` and `offset_open_curve_outward()` functions produce CW winding by default. For interior pockets, reverse with `list(reversed(verts))` after offsetting.

---

## G2/G3 ARC COMMANDS — RADIUS CONSTRAINT

**Confirmed: GRBL requires `distance(start, center) == distance(end, center)` for every G2/G3 arc.** If the start-to-center radius and end-to-center radius differ by more than a few microns, GRBL throws error 33 and halts execution. The spindle keeps running since M5 is never reached. The machine appears to "just spin" after completing whatever commands preceded the bad arc.

**Why this happens with fitted arcs:** When arc-fitting linearized outline points, the circle center is computed from 3 sample points. The arc endpoint is another sample point that may not lie exactly on that circle. The I/J offset (computed from the start point to the center) defines a radius that doesn't match the endpoint's distance to the center. The mismatch is small (typically 0.01-0.1mm) but fatal.

**The fix — endpoint projection:** After computing the circle center and I/J offset, project the arc endpoint onto the circle at exactly `r = hypot(I, J)` from the center. This adjusts the endpoint by at most the arc fitting tolerance (0.1mm) and guarantees exact radius match.

**The fix — position tracking:** The emitter must track the actual machine position through projected endpoints. Each arc's start point is the previous arc's projected endpoint, not the original input point. I/J offsets must be computed from the tracked position, not from the input point list. Without this, the radius mismatch cascades through consecutive arcs.

**Standing rule:** Every generated NC file must pass a radius consistency check before being delivered. The generator should validate all G2/G3 commands and report any with `abs(r_start - r_end) > 0.002mm`.

**The failure that prompted this rule:** March 2026 — a headstock outline cut file with 271 arc commands having radius mismatches of 0.01-0.1mm. The holes (exact semicircle arcs) cut successfully, then the first outline arc triggered error 33. The machine raised to safe Z and spun with no XY movement until manually stopped.

---

## CUT DIRECTION BY MATERIAL

Cut direction is material-dependent. Default is conventional. Switching cut direction without confirming geometry produces tear-out or climb-cut chatter on the wrong material -- confirm geometry permits it before switching.

**MDF (current primary material):**
- All passes: conventional cut
- Conventional = counterclockwise around outside profiles, clockwise inside pockets

**Solid hardwood (basswood, alder, limba — when in use):**
- Rough passes: climb cut for efficient material removal
- Finish passes: conventional cut for surface quality
- Climb = clockwise around outside profiles, counterclockwise inside pockets

When a confirmed cut file has been tested on a specific material, sub-label the file:
```
jhg_neck_pocket_wall_p40_mdf.nc
jhg_neck_pocket_wall_p40_basswood.nc
```
This preserves the known-good result for that material without overwriting the base file.

---

## FEED RATE HIERARCHY

```
Rough passes:       1500 mm/min
Penultimate passes: 1100 mm/min
Final passes:        800 mm/min
Spring passes:       750 mm/min
EMG corner ramp:     600 mm/min
Helix ramp:         1200 mm/min
Plunge (all):        500 mm/min
```

---

## FINISH PASS SEQUENCE

Accounts for bit deflection at full depth.

**Rough passes — 1500 mm/min:**
- Clear material to 1.0mm proud of target wall
- Conventional cut for MDF, climb for hardwood rough

**Penultimate passes — go and come at 1100 mm/min:**
- Advance to 0.3mm proud of target (removing 0.7mm from rough leave)
- At path end: do not retract — reverse direction at cutting depth
- Return along same path in opposite direction
- Two passes total, opposite directions, same depth

**Final passes — go and come at 800 mm/min:**
- Advance to true target wall (removing remaining 0.3mm)
- At path end: do not retract — reverse direction at cutting depth
- Return along same path in opposite direction
- Two passes total, opposite directions, same depth

---

## REVERSAL OFFSET RULE

Every pass must begin and end beyond the reversal points of the previous pass, extended along the path direction only. No two passes in any sequence share a reversal or entry/exit point.

- The overshoot must never deviate from the path into the workpiece geometry
- At corners or curved terminations, the overshoot follows the contour
- If geometry does not permit a safe overshoot in the path direction, flag it and ask before generating

Use parameter `PASS_OVERSHOOT` (default 3mm), incrementing per pass:
```
Penultimate: overshoots path ends by 1 × PASS_OVERSHOOT (3mm)
Final:       overshoots path ends by 2 × PASS_OVERSHOOT (6mm)
```

**Scope — applies to:**
- Multi-pass wall and finish sequences
- Multi-depth clearance paths

**Scope — does not apply to:**
- Drill holes and small diameter circles — separate logic
- Single-pass operations — case by case
- Ramp entries — subset of hole drilling logic, handled separately

---

## GO-AND-COME ARC REVERSAL

When reversing arc-fitted commands for a go-and-come return pass, **never flatten arcs to G1**. Properly reverse each arc: swap G2↔G3 and recompute I/J from the new start point (which was the old endpoint) to the same circle center.

**Algorithm:** For each arc command in reversed order:
1. The reversed arc's start point = the forward arc's endpoint
2. The reversed arc's endpoint = the forward arc's start point
3. Compute new I/J = (center_x - rev_start_x, center_y - rev_start_y)
4. Swap direction: G2 becomes G3, G3 becomes G2
5. The circle center stays the same in absolute coordinates

**The `reverse_commands()` function** takes the forward command list and the forward path's start point. It walks backwards through the point chain, recomputing I/J for each arc from its new start position. G1 commands simply reverse order with swapped endpoints.

**The failure that prompted this rule:** March 2026 Panel A test cut — the go-and-come return pass converted all G2/G3 arcs to single G1 segments. At horn tips, a smooth arc spanning 10mm of curve became one straight line, gouging across the curve and leaving visible flat facets. After implementing proper arc reversal: 138 arcs recovered, horn surface smooth in both directions.

---

## GO-AND-COME POLYGON REVERSAL -- SIDE INDEX

When reversing a polygon's vertex list for a go-and-come return pass, **do not re-derive the plunge side** by calling a side-ranking function (e.g., `find_longside_plunge`) on the reversed list. Reversing a vertex list changes which vertices are paired as "sides," so a side-ranking function will pick a different side than the forward pass used. The return pass then starts from the wrong corner and cuts diagonally across the pocket.

**The fix:** Compute the reversed side index arithmetically from the forward index:

```
rev_side_idx = (n - 2 - forward_side_idx) % n
```

where `n` is the vertex count and `forward_side_idx` is the side the forward pass plunged on. This maps the same geometric edge (module) from the forward vertex ordering to the reversed ordering.

**The plunge point coordinates do not change** -- the return pass starts and ends at the same physical location as the forward pass. Only the corner traversal sequence needs the corrected index.

**Example failure:** Panel A/C EMG ear cavities -- the return pass jumped ~41mm diagonally across the rectangle at full depth instead of retracing the forward path. The diagonal was invisible in the verification SVG because the overlay only draws polygon outlines, not traversal order.

---

## END-OF-PROGRAM RETRACT

End of program retracts to SAFE_Z (12mm), then homes to X0 Y0. PARK_Z crashed the Z limit when used at end of program -- it is for mid-job bit swaps only, and must be set to clear bit length + stock height + workholding for that specific job. It is not a constant.

```gcode
G0 Z12.00     ; safe Z — NOT Z50
M5            ; spindle off
G0 X0 Y0      ; return to origin
M2            ; program end
```

---

## EMG APERTURE CORNER FEED RAMP

Rectangular pickup cavities (EMG apertures) use a corner feed ramp to prevent overtravel/bite at 90° corners. The pattern for each corner:

1. **Approach:** Straight segment at nominal feed (FEED_ROUGH, FEED_PENULT, etc.)
2. **Ramp down:** At CORNER_RAMP_MM (5.0mm) before the corner vertex, feed drops to FEED_CORNER (600mm/min)
3. **Corner vertex:** The tool traverses through the actual corner at FEED_CORNER
4. **Ramp up:** At CORNER_RAMP_MM after the corner, still at FEED_CORNER
5. **Resume:** Next straight segment returns to nominal feed

**The corner vertex is always present in the G-code.** The ramp inserts intermediate points on the sides, but the 90° turn happens at the actual polygon vertex. Do not skip the vertex — that creates a chamfer.

**Parameters:**
```
FEED_CORNER    = 600      # mm/min
CORNER_RAMP_MM = 5.0      # mm — distance before/after corner
```

---

## PLUNGE POINT PLACEMENT

Plunge points land on the **rough offset path**, not the finish path. Finish passes erase the entry witness.

Place the plunge on a straight segment, stood off from the nearest corner or direction change by at least `CORNER_RAMP_MM`. This gives the bit a stable feed run before its first direction change. If geometry doesn't permit full standoff, plunge at the best available position and flag it in the NC header.

**SVG convention:** Mark each plunge point with a circle at bit diameter and an inner X cross. Group all plunge markers in their own `<g>` for convenient selection in Illustrator.

---

## ARC-FIT PATH ROTATION

`arc_fit_points()` processes a point list from index 0 to index N-1. If the plunge point is not at index 0, the first arc-fitted command targets a point far from where the machine is positioned after plunging, and the machine cuts a straight line across the workpiece interior to catch up.

**Standing rule:** Before calling `arc_fit_points()`, rotate the point list so the plunge point is at index 0. Use:

```python
def rotate_closed_path(pts, start_idx):
    return pts[start_idx:] + pts[:start_idx]
```

Apply this to all path variants (rough, penult, finish) using each variant's own closest-point-to-plunge index. The rotation must happen after offset computation but before arc fitting.

**See also:** G2/G3 ARC COMMANDS -- the arc fitter assumes index 0 is the start point; this rule ensures that assumption is always satisfied.

**Example failure:** Panel C body outline -- the SVG path's M command started near the bottom of the body, but the plunge point was near the top. The first cutting move after plunge was a straight line across the entire body interior (~400mm) to reach where the arc-fitted command list thought it was starting.

---

## Z COORDINATE FORMAT

All Z values in G-code use two decimal places (`.2f`). This supports fractional depths like DIP_DEPTH=1.25mm. Using `.1f` truncates 1.25 to 1.2 — a 0.05mm error that accumulates over multiple operations.

Applies to both `G0 Z` (rapid) and `G1 Z` (linear) commands.

---

## TOOL PATH EFFICIENCY

Stay at cutting depth and traverse directly between positions. Safe Z is for plunge entries, plunge exits, and end of program only. Unnecessary safe Z retracts are inefficient -- raise only when changing plunge point or geometry requires it (obstacle, workholding).

---

## INTERLEAVED ROUGH -- STAY AT DEPTH

When pocket and outline rough passes are interleaved by Z level, the full cycle at each depth is one continuous loop with no lift:

```
plunge at pocket start -> pocket pass -> outline pass -> back to pocket plunge -> plunge to next depth -> ... -> safe-Z retract
```

The transitions between pocket and outline are short traverses (typically 2--3mm where the paths nearly meet). These traverse at cutting feed, not as rapids. Lifting between pocket and outline within the same Z level frays the pocket exit point and wastes time replunging to the same depth.

Safe-Z retract happens once, after all Z levels are complete.

**Note:** The interleaved penultimate, final, and spring passes already follow this pattern (continuous perimeter pass with no mid-pass retract). This rule ensures rough passes match.

**Example failure:** Panel A interleaved rough -- every pocket->outline and outline->pocket transition lifted to Z12, replunged to the same depth, and frayed the neck pocket exit spike on every retract through 11 depth levels.

---

## STANDARD HEADER BLOCK

Every generated NC file must begin with this header before the modal setup block:

```gcode
; =============================================
; [File description — what this file does]
; Machine:   TwoTrees TTC450 PRO
; Bit:       [size and type]
; Source:    [parent .py generator file]
; Origin:    Bounding box center = XY0
; Material:  [if material-specific]
; [Key parameters]
; Generated: [date]
; =============================================
```

---

## PIPELINE DEFAULTS

- Always generate a verification SVG alongside every NC file
- **Verification overlays must come from the generator.** The generator outputs its computed toolpaths (rough, penultimate, finish, spring offsets) back onto the SVG as lime green dashed overlays, converted from NC mm to SVG px via the inverse coordinate transform. The overlays represent exactly what the NC file will cut. Hand-computed overlays using a separate code path will not catch generator errors. *See claudecam workflow doc, Generator Toolpath Overlays section.*
- Run order must be stated explicitly when multiple files are part of one operation
- **When a re-uploaded verification SVG contains a `<i:aipgf>` block inside `<metadata>`:** strip the entire `<metadata>` block before making any edits. Illustrator embeds PGF data on save -- if not stripped, Claude's edits will be invisible when Jason reopens the file in Illustrator.

---

## BIT SWAP PROTOCOL

When a job requires multiple bits:
- File 1 runs and returns to Z12 — never park at Z50+ between files
- Operator swaps bit
- Operator re-zeros Z on stock surface
- File 2 runs
- State this explicitly in each file's header
