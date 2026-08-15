# JHG G-Code Hygiene
Jason Hunter Guitars — TwoTrees TTC450 PRO Standards
*These are the standing rules for every G-code file generated for JHG. Read this before writing or editing any NC file.*

**1.8 → 1.9 (2026-08-14):** Recovered four emission-methodology sections (G-Code Command Selection, Arc Fitting Constraints, Helical Orbit, Point Cleanup) that were lost in the shop-file-standards 1.7→1.8 halving — this doc is now their home, making it the G-code authority. Rewrote FINISH PASS SEQUENCE to the shipped five-stage ladder (stepped penultimate + spring pass), with parameter names in prose and the generator PARAMETERS block as the canonical value source. Helical Orbit corrected to shipped practice: single orbit radius (direction reversal, not a diameter step, is the deflection-correction mechanism) and the floor-flattening loop added as its own step. Value history and verification: `Evolution/PARAM_CENSUS.md` (local archive).

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

## G-CODE COMMAND SELECTION

*Recovered 2026-08-14 from `jhg_shop_file_standards_1_7.md` (lost in the 1.7→1.8 halving).*

**G2/G3 (circular arcs) — preferred for:**
- Tuner holes and any circular geometry — full circles, semicircles, helical ramps
- Any curve segment where a circular arc fits within tolerance
- Long sweeping profile curves that can be approximated by tangent-continuous biarc sequences

**G1 (linear segments) — used when:**
- Geometry is not well-approximated by circular arcs (inflection points, compound S-curves, tight horn tips)
- Biarc fitting would require more arcs than equivalent G1 segments
- Tolerance math is uncertain — G1 at known tolerance is safer than a bad arc fit

**Linearization tolerance (when G1 is used):**
- `0.05mm` — tight, for horn tips and features where shape fidelity matters on final wood
- `0.1mm` — standard, good default for most profile work
- `0.2mm` — coarse, acceptable for MDF templates and roughing passes

---

## ARC FITTING CONSTRAINTS

*Recovered 2026-08-14 from `jhg_shop_file_standards_1_7.md` (lost in the 1.7→1.8 halving). The four parameters themselves remain a PARAMETERS-block requirement in Shop File Standards; this section is the reasoning.*

When fitting G2/G3 arcs to linearized outline points, the fitter must be constrained. Greedy "fit the longest possible arc" produces bad results at transitions and inflection points where curvature changes direction or magnitude. These constraints are not optional:

```python
ARC_FIT_TOL   = 0.1    # mm — max deviation from fitted arc to accept it
ARC_MAX_CHORD = 15.0   # mm — max straight-line distance for a single arc
ARC_MIN_R     = 1.0    # mm — arcs tighter than this are likely noise
ARC_MAX_R     = 200.0  # mm — arcs larger than this are nearly straight, use G1
```

*(Values illustrative — headstocks ship `ARC_MAX_R = 200.0`, body panels ship `100.0`. Canonical per job in each generator's PARAMETERS block.)*

**Rules:**
- Max chord length limits how far any single arc can reach. Without this, the fitter merges long sweeps into single arcs that cut across the actual curve between sample points.
- Max radius excludes near-straight segments where G1 is equally accurate and simpler.
- Min radius excludes spurious tiny arcs from noise in the point data.
- Direction consistency: if extending an arc would flip CW/CCW, the arc must stop. An inflection point is not a single arc.
- Sweep angle must stay under ~170 degrees. GRBL handles full circles for holes but profile arcs should be shorter segments.

**The failure mode this prevents:** March 2026 — a greedy arc fitter with no chord limit merged 6+ points into single arcs spanning 24mm. The arcs passed the radial tolerance check (all points were within 0.1mm of the fitted circle) but the arc path between those points deviated significantly from the actual curve. The result was visibly wrong at the top-left corner and right-side S-curve of the headstock outline.

**See also:** the next section — every fitted arc must also satisfy GRBL's endpoint-radius constraint before it is emitted.

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
Rough passes:         1500 mm/min
Penultimate passes:   1100 mm/min
Final passes:          800 mm/min
Spring passes:         750 mm/min  (body-panel outline; neo4 ships 669 for all its spring laps)
Hole spring laps:      669 mm/min  (helical orbit spring — see Hole Cutting Strategy)
EMG corner ramp:       600 mm/min
Helix ramp:           1200 mm/min  (neo4; neo3 still ships 1500 — Gen-0 sediment)
Plunge (all):          500 mm/min
```

Values are illustrative of current shipped practice; canonical per job in each generator's PARAMETERS block. **Naming caveat:** the fleet is inconsistent — body panels use `FEED_SPRING` for the outline spring (they have no holes); headstock generators use `FEED_SPRING` for *hole* spring laps and `FEED_OUTLINE_SPRING` for the outline. Flagged for code cleanup; until then, check which one a given generator means before citing it.

---

## FINISH PASS SEQUENCE — FIVE STAGES

**The objective:** walls that come off the machine smooth enough that sanding stays light. The multipass technique inherently leaves banding at Z-pass boundaries ("cake layers" — stepdown witness marks; see the vocabulary table in the troubleshooting doc), and a tool cutting sideways at full depth is loaded against its least rigid axis — too much material engagement and it flexes away, springs back, and writes that into the wall. Every stage below exists to control that lateral load. The leave values are the load control.

**Canonical values live in each generator's PARAMETERS block, not in this doc.** The current ladder (Panel A `jhg_body_pnlA_rot17_8_1.py`, adopted by Panel C's stepped lineage the same session) is `ROUGH_LEAVE = 2.0`, `PENULT_LEAVE_1 = 1.15`, `PENULT_LEAVE = 0.3`. Numbers appearing below are illustrative of that ladder, not authoritative.

**Stage 1 — Rough passes (`FEED_ROUGH`):**
- Clear material to `ROUGH_LEAVE` proud of target wall, stepping down in `PASS_DEPTH` increments
- Conventional cut for MDF, climb for hardwood rough

**Stage 2 — Penultimate step 1 (`FEED_PENULT`):**
- Go-and-come at full depth on the `PENULT_LEAVE_1` (midpoint) offset — trace the contour, reverse at cutting depth without retracting, return along the same path
- No lift from rough: traverse at depth to the penult start
- Splitting the penultimate into two bites keeps the lateral load per pass low enough that the tool doesn't deflect and re-band the wall it's there to smooth

**Stage 3 — Penultimate step 2 (`FEED_PENULT`):**
- Go-and-come at full depth on the `PENULT_LEAVE` (final leave) offset

**Stage 4 — Final passes (`FEED_FINISH`):**
- Go-and-come at full depth on the true target wall (0mm leave)
- This is the cut that defines the finished surface

**Stage 5 — Spring pass (`FEED_SPRING`):**
- Two full laps at final geometry, near-zero cutting load — lets the bit spring back from residual deflection, equalizing the wall
- **Standing practice, not optional:** every shipped generator emits it (Neo 4 verified NC "HOLES FIRST -> rough -> penultimate -> final -> spring"; Panel A 8_1 `SPRING: POCKET + BODY OUTLINE`; every Panel C lineage `OUTLINE: SPRING`)

**Causation labels on the current ladder:**
- **Confirmed:** the earlier 1.0 / 0.3 single-step ladder is insufficient on MDF — cake layer banding survived it (the failure that raised the leaves and split the penultimate)
- **Confirmed:** 2.0 / 1.15 / 0.3 is what the current generators run
- **Conjecture (open):** that 2.0mm is *sufficient* — the verification cut aborted before the wall that would have answered it. Cheap to settle on the next cut; wrong-leave cost is sanding time, not part dimensions (finish always runs at true target)
- Leave requirements are **material-dependent** — MDF needs more leave than initially estimated; hardwood values unestablished

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

## HOLE CUTTING STRATEGY — HELICAL ORBIT

*Recovered 2026-08-14 from `jhg_shop_file_standards_1_7.md` (lost in the 1.7→1.8 halving) and corrected against the shipped implementation (`jhg_headstock_neo4_template_1_1.py` + `jhg_headstock_neo4_verified_3.nc`, verified step-for-step). Where the archived spec and the shipped NC disagreed, the NC won.*

All tuner holes and round holes use the same strategy. Each hole is completed to full depth before moving to the next hole. Holes are cut before the outline.

**Execution order within each hole:**

1. **Rapid to start position** — G0 to safe Z, then G0 to a point on the helix circle
2. **Plunge to stock surface** — G1 Z0.5 at `PLUNGE` feed (approach from above)
3. **Helical ramp to depth** — G2 arcs with simultaneous Z descent at `HELIX_R`, removing the bulk of the material. Z descent per revolution set by `HELIX_PITCH`
4. **Floor-flattening loop** — one full closing loop at final depth at `HELIX_R`. A helical ramp leaves a spiral floor by geometry; this loop flattens it. Omit it and the hole has a stepped bottom — harmless for a through-bore, wrong for anything seating flat
5. **Finish orbit 1 (CCW)** — two full laps at `ORBIT_R_FINISH`, `FEED_FINISH`. Removes remaining stock; the bit deflects outward (away from the hole wall)
6. **Finish orbit 2 (CW)** — two full laps at the **same** `ORBIT_R_FINISH`, opposite direction. The deflection correction comes from the **direction reversal at constant radius** — same path, opposite direction, deflection cancels. This is the go-and-come principle applied to a circle; the outline's penultimate/final passes are the same mechanism on an open contour. (The archived 1_7 spec parameterized two separate finish diameters, implying a diameter-step correction — no shipped generator has ever had a second orbit diameter. The mechanism is the reversal.)
7. **Spring pass (CCW)** — two full laps at `ORBIT_R_FINISH`, `FEED_SPRING` (669 mm/min for holes). Near-zero load; the bit springs back from residual deflection, equalizing the hole
8. **Retract to safe Z** — G0 Z12

**Hole-to-hole travel:** Retract to safe Z between holes. Cut order may be optimized to minimize total travel, but each hole runs the full sequence above before the next hole begins.

**Key parameters per hole (shipped form):**

```python
ORBIT_R_FINISH = HOLE_R - BIT_R          # single finish/spring orbit radius
HELIX_R        = ORBIT_R_FINISH - 0.5    # ramp radius — leaves 0.5mm for the finish orbits
HELIX_PITCH    = 0.9                     # mm Z descent per revolution
FEED_HELIX     = 1200                    # mm/min
FEED_SPRING    = 669                     # mm/min — hole spring laps
```

**Why G2/G3 for holes:** A circle is exactly what G2/G3 describes. One G2 command replaces 36+ G1 segments per revolution. The controller interpolates the arc internally at `$12` tolerance (default 0.002mm), keeping motion perfectly smooth. File size drops dramatically — a six-hole operation that was 10,000+ lines of G1 becomes a few hundred lines.

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

**Parameters (illustrative — canonical per job):**
```
FEED_CORNER    = 600      # mm/min
CORNER_RAMP_MM = 5.0      # mm — distance before/after corner (Panel A shipped value)
```

**Short-side geometric constraint:** the ramp distance must fit the side. Panel C's stepped lineage reduced `CORNER_RAMP_MM` to 3.0 *and* capped it per-segment with `ramp_len = min(CORNER_RAMP_MM, 0.4 * seg_len)` — at 5.0, the ramp zones from adjacent corners of a 7.61mm EMG-ear short side overlap and no nominal-feed run survives between them. Any generator with sides shorter than `2 × CORNER_RAMP_MM` needs the cap, not just a smaller constant.

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

**See also:** G2/G3 ARC COMMANDS — RADIUS CONSTRAINT above -- the arc fitter assumes index 0 is the start point; this rule ensures that assumption is always satisfied.

**Example failure:** Panel C body outline -- the SVG path's M command started near the bottom of the body, but the plunge point was near the top. The first cutting move after plunge was a straight line across the entire body interior (~400mm) to reach where the arc-fitted command list thought it was starting.

**Provenance (established 2026-08-15).** The defect was **introduced**, not inherited. A scan of every archived Panel C NC for long cutting moves at depth puts it precisely: the 2026-03-19 generation is clean; the chord first appears on **2026-03-23**, in the session that synced Panel C's leave ladder to current shop standards *using Panel A as the reference*. Panel A's structure was grafted onto Panel C, and the plunge marker and the path's start index stopped agreeing. It persisted into `jhg_body_pnlC_rot17_2.nc` (2026-03-25 17:11) — the file run **that same day**, where a rapid to the heel, a plunge to Z-16, and a 399mm chord across the body triggered an emergency stop (not a jam; nothing stalled) — and was cleared in April only as a side effect of the Big Ears rebuild. Full entry: `transition/jhg_archaeology_plan_1_0.md`. **The lesson is the pattern, not the bug:** a quality-improvement pass introduced a machine-safety defect. The fix shipped the same day — `Proven Cuts/Panel C proven/jhg_body_pnlC_rot17_3.nc` (17:31, verified 2026-08-15: zero long moves at depth, passes `verify_nc.py`) is the corrected file, 20 minutes after the flawed 17:11 one. It was simply never cut; shop time ran out that evening and no Panel C cut happened again. So the rule was written, the fix was made, and the loop that broke was **verifying the rule against the file that actually ships** — which is exactly what a gate closes. Regenerate and re-verify after any cross-panel sync.

**Status: CONFIRMED FIXED 2026-08-13.** `jhg_body_pnlC_rot17.py`'s `emit_body_outline()` now rotates `rough_pts`/`penult_pts`/`finish_points_mm` to each pass's own plunge index before calling `arc_fit_points()`, matching this rule. Verified by diffing the regenerated `.nc` against the prior output -- the chord jump is gone from all three passes (rough, penultimate, final). This rule existed here before the fix was made; if a similar symptom shows up in another generator, check this section before re-diagnosing from scratch. See CHECK EXISTING RULES BEFORE DEBUGGING in the troubleshooting doc.

---

## POINT CLEANUP RULES — KEEP NARROW, DO NOT CASCADE

*Recovered 2026-08-14 from `jhg_shop_file_standards_1_7.md` (lost in the 1.7→1.8 halving).*

When cleaning up linearized points (e.g. removing degenerate bezier artifacts), cleanup rules must:

- Target a **specific failure mode**, not a general pattern
- Run in a **single pass** — never use `while changed` iterative cleanup, which cascades through smooth curves eating legitimate points
- Use **narrow thresholds** tied to the specific artifact being removed
- Verify they don't affect points far from the target area

**The failure mode this prevents:** March 2026 — a collinear micro-segment removal rule (turn angle < 5 deg, segment < 2mm) was designed to fix one degenerate bezier corner. The iterative loop cascaded through the lower-right curve of the headstock, removing 14 real geometry points and flattening the curve. The rule was correct for its target but too broad for general application.

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

**Interleaved finishing stages follow the same no-lift principle** as one continuous perimeter pass per stage: `pocket forward -> outline forward -> outline reverse -> pocket reverse`. This ordering applies to the penultimate steps, the final pass, and the spring pass (spring runs the perimeter as two full laps at spring feed).

**Example failure:** Panel A interleaved rough -- every pocket->outline and outline->pocket transition lifted to Z12, replunged to the same depth, and frayed the neck pocket exit spike on every retract through 11 depth levels.

---

## FILE SIZE SMOKE TEST

*Salvaged 2026-08-14 from `jhg_shop_file_standards_1_7.md` — retired as a spec, kept as a check.*

With G2/G3 holes and constrained arc fitting, a complete six-tuner-hole headstock file lands under ~4,000 lines (holes ~200-400, outline 1,000-3,000). A headstock NC **well over 4,000 lines means the arc fitter didn't engage** and the file has silently degraded to all-G1 (~14,000+ lines) — check the arc fitting parameters and the fitter's input before delivering. This is a smoke test, not a target: a legitimately complex file can exceed it, but an unexplained size jump is the cheapest possible defect signal.

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
- **Verification overlays must come from the generator.** The generator outputs its computed toolpaths (rough, penultimate, finish, spring offsets) back onto the SVG as lime green dashed overlays, converted from NC mm to SVG px via the inverse coordinate transform. The overlays represent exactly what the NC file will cut. Hand-computed overlays using a separate code path will not catch generator errors. *See [claudecam workflow doc](jhg_claudecam_workflow_1_1.md), Overlay Limitation section — overlays verify where the bit cuts, not traversal order.*
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
