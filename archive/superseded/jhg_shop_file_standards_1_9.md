# JHG Shop File Library Conventions
Jason Hunter Guitars — Cross-File Standards
*These conventions apply to every file type in the JHG library: G-code, Python generators, OpenSCAD models, and SVGs. Read this before creating, editing, or importing any JHG file.*

**1.8 → 1.9 (2026-08-13):** Added Bezier Sample Density and Offset Method subsections under SVG-TO-G-CODE METHODOLOGY, recovered from a 52-conversation audit of pre-Claude-Code sessions that never made it into the tracked docs (`jhg_orphaned_knowledge_1_1.md` in `Transition Support Docs/`).

---

## THE CORE PRINCIPLE

**The `.py` generator is the deliverable. Everything else is an output.**

Every NC file, SVG, and visual confirm is downstream of a Python model. When you have the `.py`, you have the entire logical model — all geometry, all parameters, all relationships. Without it, you have a collection of output files with no explanation of how they relate to each other.

- Always upload the `.py` first in any new session
- Always reference the source `.py` in every output file header
- When a parameter changes, change it in the `.py` and regenerate — do not patch output files directly unless the change is explicitly a one-off test

---

## FILE PAIRING CONVENTION

Files travel in pairs or sets. The naming convention makes the relationship explicit:

```
jhg_neck_pocket_wall.py           ← source model
jhg_neck_pocket_wall_p40.nc       ← G-code output
jhg_neck_pocket_wall_p40.svg      ← verification SVG
```

For multi-bit jobs, files share a base name with a suffix indicating the bit:

```
jhg_body_outline_rot17_1-4.nc     ← 1/4" bit
jhg_body_outline_rot17_1-8.nc     ← 1/8" bit
```

When a file has been confirmed on a specific material, sub-label it:

```
jhg_neck_pocket_wall_p40_mdf.nc       ← confirmed on MDF
jhg_neck_pocket_wall_p40_basswood.nc  ← confirmed on basswood
```

The base file (no material suffix) remains the starting point. Material-specific files are confirmed derivatives. Never overwrite the base file with a material-specific result.

---

## SECTION LABELING SYSTEM

All JHG files use a consistent section labeling system. This is how the library is navigated, how sections are extracted for independent work, and how they are re-imported. A Claude working on any JHG file can locate, extract, modify, and re-import any section using this system.

### Standard Section Header

```
; =============================================
; SECTION: [NAME IN CAPS]
; Description: [what this section does]
; Source:      [parent .py file if applicable]
; Parameters:  [key values driving this section]
; Depends on:  [sections that must precede this]
; =============================================
```

### Current Section Name Library

Use these names when they fit. If none apply, ask Jason and propose a new name before inventing one. New confirmed names get added here so the library stays consistent.

| Name | Purpose |
|------|---------|
| `PARAMETERS` | All tunable values — always at top of file |
| `MACHINE SETUP` | Modal defaults, spindle on, safe Z |
| `CLEARING` | Bulk interior material removal — may use variants e.g. `CLEARING: SPIRAL`, `CLEARING: RASTER` |
| `ROUGH PASSES` | Stepped depth passes with offset clearance |
| `PENULTIMATE PASSES` | Pre-finish go-and-come, 0.2mm proud of target |
| `FINISH PASSES` | Final go-and-come to true target wall |
| `DRILL/HOLES` | Point drilling operations — separate logic from wall passes |
| `HOLES: HELICAL ORBIT` | Helical ramp + orbit strategy using G2/G3 arcs — see SVG-to-G-Code Methodology |
| `OUTLINE: ROUGH` | Profile roughing passes at ROUGH_LEAVE offset from target |
| `OUTLINE: PENULTIMATE` | Profile penultimate pass at PENULT_LEAVE offset |
| `OUTLINE: FINAL` | Profile final pass at true target geometry |
| `OUTLINE: SPRING` | Near-zero-load cleanup pass at final geometry — deflection equalization |
| `INTERLEAVED ROUGH: POCKET + OUTLINE` | Pocket then outline rough at each Z level — continuous leveling |
| `INTERLEAVED PENULTIMATE: POCKET + OUTLINE` | Pocket then outline penult as one continuous perimeter pass |
| `INTERLEAVED FINAL: POCKET + OUTLINE` | Pocket then outline final as one continuous perimeter pass |
| `INTERLEAVED SPRING: POCKET + OUTLINE` | Two full perimeter laps (pocket + outline) at spring feed |
| `RETRACT AND PARK` | End of program: safe Z, spindle off, home. PARK_Z for mid-job bit swaps only -- must be tuned per job. |

When proposing a new section name, consider whether it is a variant of an existing name (use colon syntax) or genuinely new. Either is fine — just confirm before adding.

### Rules for Section Use

- Every file must have a `PARAMETERS` section at the top
- Every section must include at minimum a name and a `Depends on:` field
- When extracting a section for independent work, preserve the full header
- When re-importing a section, verify `Depends on:` before placement — some sections assume modal state set by a preceding section
- Do not invent section names mid-session without updating this doc

---

## PARAMETERS BLOCK STANDARD

The `PARAMETERS` section must be the first thing in every file after the file header. All tunable values live here. Nothing that can be parameterized should be hardcoded deeper in the file.

**In Python generators:**
```python
# =============================================
# SECTION: PARAMETERS
# =============================================
# SVG output scale — always define when the generator produces SVG
SVG_SCALE        = 2.8346  # px/mm - set so SVG coords match real geometry
#                            If targeting mm-native SVG, set SVG_SCALE = 1.0
#                            and size the viewBox to real-world mm dimensions
BIT_R            = 3.175   # mm - 1/4" upcut spiral
WALL_OFFSET      = 0.4     # mm - positive = looser pocket
EMG_OFFSET       = 1.0     # mm - pickup cavity enlargement (stacks on WALL_OFFSET)
ROUGH_LEAVE      = 2.0     # mm - proud of target after rough passes
PENULT_LEAVE     = 0.5     # mm - proud of target after penultimate passes
PASS_OVERSHOOT   = 3.0     # mm - reversal offset increment per pass
DEPTH_TOTAL      = 16.0    # mm - total cut depth
PASS_DEPTH       = 1.5     # mm - max depth per pass
FEED_ROUGH       = 1500    # mm/min
FEED_PENULT      = 1100    # mm/min
FEED_FINISH      = 800     # mm/min
FEED_SPRING      = 750     # mm/min
FEED_CORNER      = 600     # mm/min — EMG aperture corner approach/exit
CORNER_RAMP_MM   = 5.0     # mm — distance before/after corner to begin slowdown
FEED_HELIX       = 1200    # mm/min (helical ramp into holes)
PLUNGE           = 500     # mm/min
RPM              = 18000
SAFE_Z           = 12.0    # mm
# PARK_Z is not used for end-of-program retract. End-of-program: safe Z then home.
```

**In G-code files:**
```gcode
; =============================================
; SECTION: PARAMETERS
; =============================================
; BIT_R:          3.175mm  (1/4" upcut spiral)
; WALL_OFFSET:    0.4mm    (positive = looser)
; EMG_OFFSET:     1.0mm    (pickup cavity enlargement, stacks on WALL_OFFSET)
; ROUGH_LEAVE:    2.0mm
; PENULT_LEAVE:   0.5mm
; PASS_OVERSHOOT: 3.0mm
; DEPTH_TOTAL:    16.0mm
; PASS_DEPTH:     1.5mm
; FEED_ROUGH:     1500 mm/min
; FEED_PENULT:    1100 mm/min
; FEED_FINISH:    800 mm/min
; FEED_SPRING:    750 mm/min
; FEED_CORNER:    600 mm/min  (EMG corner approach)
; CORNER_RAMP_MM: 5.0mm  (ramp distance before/after corners)
; FEED_HELIX:     1200 mm/min  (helical ramp into holes)
; PLUNGE:         500 mm/min
; RPM:            18000
; SAFE_Z:         12.0mm
; =============================================
```

---

## SVG DIAGRAM WORKFLOW

Use the SVG artifact workflow for iterative diagrams -- not the Python render pipeline. Run Python once to compute geometry, save as `.svg`, iterate on the SVG directly from that point. Re-run Python only when a parameter value changes that affects computed coordinates.

*Full workflow, visualization hierarchy, Illustrator round-trip rules, and bezier toolpath pipeline: [jhg_claudecam_workflow_1_0.md](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_claudecam_workflow_1_0.md)*

### SVG Source Reference

Every SVG generated from a Python model must include this in its opening comment:

```xml
<!-- source: [filename.py] | key params: PARAM=value, PARAM=value -->
```

### SVG Real-World Scale — Mandatory

**Every JHG SVG must be scale-documented, without exception.** SVG coordinate space is unitless by default. Without an explicit scale record, the file is a drawing with no known relationship to physical reality. A future session — or a future Claude — cannot recover that relationship without a reference measurement and the math to derive it.

**Rule 1 — Prefer mm-native viewBox.**
When generating an SVG from a Python model, set the viewBox to match real-world millimeter dimensions, not pixel or arbitrary canvas dimensions. If the body is 450mm tall, the viewBox height should be 450 (or proportional). SVG coordinates then equal millimeters directly. No conversion required to interpret or edit the file.

```xml
<!-- viewBox in mm: width=325.4mm height=450.2mm -->
<svg viewBox="0 0 325.4 450.2" ...>
```

**Rule 2 — Always embed a SCALE block comment.**
Whether the file is mm-native or pixel-native, the opening comment block must include:

```xml
<!--
  JHG SVG — [description]
  Source:     [filename.py or "unknown"]
  Units:      [mm | px]
  Scale:      [px/mm factor, e.g. "2.8346 px/mm" | "1 px = 1 mm" if mm-native]
  Ref dim:    [the measurement used to establish scale, e.g. "neck pocket wall-bottom span = 55.712mm"]
  Canvas:     [e.g. "1296 x 1296 px" or "450.2 x 325.4 mm"]
  Generated:  [date]
-->
```

**Rule 3 — If the source .py is unknown, derive and document scale from a physical measurement.**
When working from an orphaned SVG with no .py file, the first task before any geometry work is to establish scale from a known real-world dimension. Document that reference measurement in the SCALE block so any future session can verify it independently.

**Rule 4 — Never leave scale as "TBD" in a file that leaves the session.**
Scale may be unresolved mid-session during exploration. It must be resolved and documented before the file is saved as an output. A file with unconfirmed scale is not a JHG deliverable.

**The failure mode this prevents:** An SVG exported from Illustrator or generated by a previous Claude carries pixel coordinates with no unit context. A subsequent session has no way to recover real-world dimensions without a physical reference measurement and the derivation work to match it. This has already happened once (Body_w_carveouts_1.svg, March 2026 — scale recovered from neck pocket wall-bottom span measurement).

### SVG + Illustrator Round-Trip -- Strip PGF Metadata

When a re-uploaded SVG contains a `<i:aipgf>` block inside `<metadata>`, strip the entire `<metadata>` block before making any edits. Illustrator embeds PGF data on save -- if not stripped, Claude's edits will be invisible when the file is reopened in Illustrator.

*Full explanation and failure history: [jhg_claudecam_workflow_1_0.md](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_claudecam_workflow_1_0.md), Illustrator Round-Trip section.*

---

## OPENSCAD METHODOLOGY

All JHG OpenSCAD models follow the **Clipped Panel** approach:

- Each panel is a self-contained module
- Raw board geometry defined inside the module
- Placement and orientation applied outside
- All miters and joints use boolean blade operations — never Pythagorean coordinate math
- Each module includes `echo()` statements for shop dimensions
- No hardcoded coordinates that could be derived from parameters

---

## SVG-TO-G-CODE METHODOLOGY

**This section describes agreed methodology -- not aspirational goals.** When a generator is built, it must implement the approach documented here. If technical constraints prevent full implementation, flag the deviation and get Jason's approval before proceeding. See IMPLEMENTATION MUST MATCH THE AGREED PLAN in [jhg_troubleshooting_and_build_discipline_1_5.md](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_troubleshooting_and_build_discipline_1_5.md).

**The standing rule: let the controller interpolate whenever geometry allows.** Feed clean bezier curves to the arc fitter. GRBL generates smoother motion from G2/G3 commands than from hundreds of pre-linearized G1 segments.

### Arc Fitting Parameters

Must appear in the PARAMETERS block of every generator that uses arc fitting:

```python
ARC_FIT_TOL   = 0.1    # mm -- max deviation from fitted arc to accept it
ARC_MAX_CHORD = 15.0   # mm -- max straight-line distance for a single arc
ARC_MIN_R     = 1.0    # mm -- arcs tighter than this are likely noise
ARC_MAX_R     = 200.0  # mm -- arcs larger than this are nearly straight, use G1
```

GRBL requires exact radius match at both endpoints of every G2/G3 arc -- a mismatch of 0.01mm triggers error 33 and halts execution. Project arc endpoints onto the fitted circle and track actual machine position through projected endpoints. *See [G-Code Hygiene doc, G2/G3 Arc Commands section](jhg_gcode_hygiene_1_9.md).*

### Bezier Sample Density -- Concave Features Need Denser Sampling

`parse_svg_bezier_to_points()` (or equivalent) takes a `samples_per_curve` parameter controlling how many points each bezier segment is linearized into before offsetting and arc-fitting. This must appear in the PARAMETERS block:

```python
SAMPLES_PER_CURVE = 60   # points per bezier segment before offset/arc-fit
```

**Confirmed: 30 is too sparse at concave features.** At a concave neck-joint dip, sparse sampling gives the arc fitter too few points to constrain the fit, and it produces a large-radius arc (seen: r=17.5mm, center *inside* the body) that bulges the bit into the part instead of hugging the wall -- shop term **"loopty-loo."** Rough and penultimate passes were unaffected in the case that surfaced this because polygon-based offsetting fed the fitter different, denser sample spacing than the bezier-derived finish path did -- only the finish pass showed the defect.

**Standing rule:** default `SAMPLES_PER_CURVE = 60`, not 30. Raise further if a concave feature still produces an oversized arc after offsetting. This is a likely contributor to degenerate `ARC_MAX_R` mismatches too -- see ARC_MAX_R note above; undersampling a curve and spanning too much geometry in one fitted arc produces both symptoms from the same root cause.

**Not yet verified against the current panel-c generator (2026-08-13):** `jhg_body_pnlC_rot17.py` is still at `samples_per_curve=30`. Whether this actually produces a visible defect depends on whether panel-c's geometry has a concave feature sharp enough to trigger it -- confirm before changing, don't change blind.

### Offset Method -- `buffer()` Clips Peninsulas

**Do not use `shapely.Polygon.buffer()` for offsetting curves with peninsula features** (guitar horns, headstock tips). `buffer()` treats the shape as a filled area; peninsulas read as concavities and get clipped, with pointed tips replaced by flat chords.

Current pipeline avoids this: **PyClipper** handles rough/penultimate offsets (passes that leave stock, where miter behavior at corners doesn't reach finished geometry), and the **finish path is read 1:1 from the Illustrator-corrected bezier curve** (see workflow doc, bezier correction algorithm) rather than computed by any offset function. This is *why* the pipeline is shaped this way -- if a future change reintroduces Shapely (e.g. `medial_axis` for MazeRunr pocket-throat-width -- see MazeRunr doc), this clipping behavior re-arms for anything Shapely touches.

**PyClipper miter setting:**
```python
MITER_LIMIT = 20   # not the pyclipper default of 10
JOIN_TYPE   = JT_MITER   # not JT_ROUND -- preserves sharp concave corners at neck-pocket junctions
```
A shallow turn angle needs a long miter spike to represent correctly -- a 6.35° turn at a lower-left horn needed a ~57mm spike; the pyclipper default `MITER_LIMIT=10` capped it at 31.75mm and truncated the corner by ~25mm. Since PyClipper now only shapes rough/penultimate stock-leaving passes (not the finish path), a miter truncation here is a rough-pass inefficiency at worst, not a part-accuracy defect -- but write the value down regardless, it took a real test cut to find.

### Coordinate Transform

Document in the `.py` PARAMETERS block:

1. `SVG_SCALE` -- from SCALE block comment, or derive from a known physical dimension
2. `mm = px / SVG_SCALE` (e.g. `/ 2.8346` for 72 DPI Illustrator exports)
3. NC origin = bounding box center, flip Y axis (SVG Y-down → NC Y-up)

The transform must be invertible: any NC coordinate can be recovered to its original SVG coordinate.

### Offset Stacking -- What's Baked Into the SVG Toolpath

The SVG toolpath is the bit center line at the finish pass position. All design offsets are baked in -- the generator reads it 1:1 and computes only the leave offsets:

- Rough pass = finish toolpath + `ROUGH_LEAVE` outward
- Penultimate pass = finish toolpath + `PENULT_LEAVE` outward
- Finish and spring passes = finish toolpath (0mm offset)

**Body outline:** `BIT_R + WALL_OFFSET` baked in
**EMG ear:** `BIT_R + WALL_OFFSET + EMG_OFFSET` baked in

`EMG_OFFSET` stacks on `WALL_OFFSET` which stacks on `BIT_R`. Applying either inside the generator as well doubles the offset.

### Offset Direction Depends on Winding -- Verify at the Call Site

The offset functions use the right-hand normal: outward on CW paths, inward on CCW paths (NC Y-up space). After any path reversal for cut direction, winding flips and "outward" functions push the wrong way.

After any path reversal, verify:
```python
assert dot(offset_vec, outward_normal) > 0, "offset direction reversed -- negate ROUGH_LEAVE"
```

*Full methodology -- G-code command selection, linearization tolerances, point cleanup rules, helical orbit strategy, profile outline strategy, bezier offset toolpath method: [jhg_claudecam_workflow_1_0.md](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_claudecam_workflow_1_0.md), Integration with G-Code Pipeline section.*

---

## MODULE REGISTRY

*Placeholder -- to be built when the generator library is sufficiently established.*

This section will catalog confirmed reusable G-code modules with their parameters, material confirmations, and doc references. Each entry will cover: what the module does, what parameters it takes, which doc has the full spec, and which materials it has been confirmed on.

---

## VERSION HISTORY CONVENTION

Output files include a version tag in the filename when iterations matter:

```
jhg_wall_offset_p40.nc          ← current working version
jhg_wall_offset_p40_pretest.nc  ← snapshot before test (see troubleshooting protocol)
jhg_wall_offset_p40_mdf.nc      ← confirmed on specific material
```

Do not use dates in filenames — use the parameter encoding to identify the version. Dates belong in the file header comment only.
