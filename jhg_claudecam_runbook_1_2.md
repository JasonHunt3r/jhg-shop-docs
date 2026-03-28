# JHG ClaudeCAM Runbook
Jason Hunter Guitars -- Claude Operational Brief
*Full methodology and reasoning: [jhg_claudecam_workflow_1_0.md](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_claudecam_workflow_1_0.md)*

---

## COLLABORATION RULES

**Scope of action = exactly what was asked.** Touching elements outside the request scope creates recovery work and erodes trust.

**Label findings explicitly.**
- **Confirmed:** Isolated test, proven cause or fix
- **Precaution:** Changed as hygiene, functional impact unverified
- **Conjecture:** Plausible theory, untested

Unlabeled speculation presented as fact is not acceptable. If something is uncertain, say so before acting.

**Inherited context:** Any context from outside the current session -- memory, past chats, handoff files -- flag it before acting on it. State what it is, where it came from, ask Jason whether to preserve, test, or discard it.

**Implementation must match the agreed plan.** Before writing any generator, G-code file, or output: re-read the relevant methodology section, verify the plan matches what's documented, and if the documented approach hits a technical wall, stop and say so. A silent downgrade wastes prior work and breaks predictability.

*Full troubleshooting protocol, handoff verification procedure, and build hygiene rules: [jhg_troubleshooting_and_build_discipline_1_4.md](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_troubleshooting_and_build_discipline_1_4.md)*

---

## ON UPLOAD -- DO THIS FIRST

1. Search for `id` containing `scale-widget`. Read `data-origin-x`, `data-origin-y`, `data-axis-angle`. Confirm against SCALE block comment.
2. Find centerline segment. Extend conceptually. Establish object frame.
3. Declare: "Scale widget at [origin], axis angle [N]°. Up in object frame is toward [direction]. Scale confirmed at [value] px/mm."
4. Search for cut path (identifiers below). Report what was found.

If no scale widget: use SCALE block comment. If neither: stop and ask.

---

## VISUALIZATION HIERARCHY

| Level | Method | When |
|-------|--------|------|
| 1 | Edit SVG directly → `present_files` | Default. Anything iterative or illustrative. |
| 2 | Python once → save SVG → iterate SVG | Geometry must be computed mathematically. |
| 3 | Python → PNG | Jason asks explicitly. State and wait for yes first. |

**Pipeline gate:** Build Python generator only after corrected bezier is verified by Jason. All bezier work happens in SVG first.

**SVG render checklist:**
```
✓  .svg extension
✓  xmlns="http://www.w3.org/2000/svg"
✓  viewBox present
✓  No external dependencies
✓  present_files, not stdout
```

**Illustrator round-trip:** Strip `<metadata>...</metadata>` from any SVG edited by both Claude and Illustrator. Illustrator rebuilds from PGF data otherwise -- Claude's edits become invisible.

**File management:** Increment version suffix on every reissue. Uploads are read-only -- copy to working dir, verify write on disk.

---

## SVG COORDINATE QUICK REF

- Y-axis down. Larger Y = further down document.
- Scale: read SCALE block. Fallback: `2.8346 px/mm` (72 DPI Illustrator, Mac, 100% scale). Stop and ask if unconfirmed.
- Outward (CW polygon, SVG space): left normal `(-dy, dx)/|d|`
- NC flips Y: CW in SVG = CCW in NC. Always state which space.

---

## SMOOTHNESS CHECK

Runs at defined stages. Flag issues, pause, get Jason's input. Report with coordinates and indices -- describe and ask. A segment that looks like an artifact may be intentional -- only Jason knows.

**Import mode** (on cutpath at Phase 1, no reference curve yet):
1. Path closure -- seam gap or flyaway
2. Continuity gaps -- endpoint mismatch >0.01px between segments
3. Duplicate points -- consecutive points within 0.001px
4. Self-intersections -- path crosses itself
5. Spikes -- turn angle >150°, check neighboring curvature before flagging
6. Extreme curvature/loops -- control point displacement >3× chord length
7. *(Experimental/advisory)* Relative segment length -- <5% of median AND no smooth continuation
8. Curvature profile -- summary report, not pass/fail

**Comparison mode** (after Phase 3, Phase 4, path reversals):
- Shoot perpendiculars from reference curve, measure where toolpath intersects
- Reference: design geometry (Phase 2/3), corrected bezier (Python toolpath check)
- Smoothness = normal deviation varies gradually. Artifact = large deviation isolated among small neighbors.

---

## PHASE 1 -- CUT PATH IDENTIFICATION

**Accepted identifiers:**
- `cutpath`, `cut_path`, `cut-path`, `CutPath`, `cut path`
- `toolpath`, `tool_path`, `ToolPath`
- `offset`, `bit_path`, `finish_path` *(lower confidence -- confirm before accepting)*

**One found:** smoothness check → winding check → rename to `[name] cutpath 01` → confirm with Jason → present SVG → Phase 2

**Multiple found:** list all with layer positions → ask Jason which to use

**None found:** report present paths → ask Jason to generate in Illustrator:
- Object → Path → Offset Path, Miter joins, Miter limit 20
- Strip `<metadata>` before re-uploading
- Must be single closed path

**Winding check:** Outside profiles = CW, interior pockets = CCW (SVG space). Reversed winding: flag, state expected winding, ask Jason to confirm. A reversed path may be intentional (cut direction) or a mistake -- only Jason can distinguish them.

---

## PHASE 2 -- MEASURE ACCURACY

1. Sample design geometry 10,000--15,000 pts → lookup table
2. Sample cutpath 500--1000 evenly-spaced pts
3. Nearest-point distance = actual offset at each location
4. Deviation = actual − target. Positive = too far. Negative = too close (cuts into part).

**Tolerances:** `<0.1mm` production ready · `0.1--0.2mm` MDF ok · `0.2--0.3mm` borderline · `>0.3mm` must correct

**Accuracy Overlay:** colored dots, group as `<g id="Accuracy Overlay 01">`, increment number if prior exists. Permanent layer, rolling hide (hide 3rd back), never stripped.
- Green `<0.1mm` · Yellow `0.1--0.2mm` · Orange `0.2--0.3mm` · Red `>0.3mm`

---

## PHASE 3 -- CORRECT CONTROL POINTS

Bernstein-weighted CP nudging. Endpoints anchored. No linearization.

**Parameters:** 40--50 iterations · 31 sample pts/segment · damping 0.12 · skip <0.08px · exclude t<0.05, t>0.95

**Achieves:** 100% within 0.1mm, worst ~0.06mm, avg ~0.02mm

**Skip:** segments <5px · sharp corners/splices · inflection points (split instead)

Run smoothness check (comparison mode) after correction before proceeding.

*[Full algorithm: jhg_claudecam_workflow_1_0.md → Phase 3](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_claudecam_workflow_1_0.md)*

---

## PHASE 4 -- SPLIT FOR DIFFERENTIAL OFFSETS

When adjacent regions need different offset values:
1. Identify transition points (sharp tangent discontinuities)
2. Split into independent open paths at each transition
3. Run Phase 2--3 on each path independently with its own target offset
4. Establish splice: extend endpoint tangents to intersection → trim/extend to splice point
5. Verify endpoint coincidence programmatically
6. Jason hand-sculpts transition in Illustrator if needed

Smoothness check (import mode) on each path after splitting and after endpoint adjustment.

---

## INTEGRATION WITH G-CODE

**Arc commands:** Let the controller interpolate. Feed clean bezier curves to arc fitter.
```python
ARC_FIT_TOL   = 0.1    # mm
ARC_MAX_CHORD = 15.0   # mm
ARC_MIN_R     = 1.0    # mm
ARC_MAX_R     = 200.0  # mm
```
GRBL radius constraint: project arc endpoints onto fitted circle. Track actual machine position through projected endpoints. Validate: `abs(r_start - r_end) < 0.002mm`.

**Coordinate transform** (document in `.py` PARAMETERS):
1. `SVG_SCALE` from SCALE block
2. `mm = px / SVG_SCALE`
3. NC origin = bounding box center, flip Y

**Offset stacking:** SVG toolpath = bit center at finish position. All design offsets baked in. Generator computes only leave offsets outward from finish toolpath.

**Winding/offset direction:** After any path reversal, verify: `dot(offset_vec, outward_normal) > 0`. If negative, negate offset amount.

*[Full methodology: Shop File Standards → SVG-to-G-Code Methodology](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_shop_file_standards_1_8.md)*

---

## LAYER STACK

| # | Layer | Name pattern | Touched by |
|---|-------|-------------|------------|
| 1 | Design geometry | (original) | Nobody |
| 2 | Cutpath | `[name] cutpath 01` | Jason (Illustrator) |
| 3 | Corrected bezier | `[name] corrected` | Claude |
| 4 | Python output | `Python Path 0N (generated)` | Python generator |

Python reads layer whose name ends in `corrected` (no following numeral).

**Corrected layer naming:** replace `cutpath` with `corrected` in cutpath name. Subsequent: append 2, 3... Colors: purple · orange · Claude picks.

**Python Path colors by slot:**
- P01: fuchsia `#ff00ff` · P02: lime green `#32cd32` · P03: sunset orange `#ff6b35` · P04: yellow `#ffe033` · beyond: Claude picks

Colors assigned by slot number, not run.

---

## VISIBILITY RULES

**Philosophy:** rolling scheme -- at any time only 2 most recent of each type visible. Nothing deleted. Verified corrected layer always visible, never hidden by Python.

**During correction (Claude):**

| State | Visible | Hidden |
|-------|---------|--------|
| corrected 1 | cutpath, C1 | -- |
| corrected 2 | cutpath, C1, C2 | -- |
| corrected 3 | cutpath, C2, C3 | C1 |
| corrected 4 | cutpath, C3, C4 | C1, C2 |

**Python handoff (first run only):** hide cutpath + oldest remaining corrected layer.

**During Python iteration:**

| State | Visible | Hidden |
|-------|---------|--------|
| P01 | corrected, P01 | cutpath, older corrected |
| P02 | corrected, P01, P02 | cutpath, older corrected |
| P03 | corrected, P02, P03 | cutpath, older corrected, P01 |
| P04 | corrected, P03, P04 | cutpath, older corrected, P01, P02 |

**Python scope:** writes new layer + hides layer 3 back + handoff hide on first run. Nothing else.

**Overlay limitation:** overlays verify *where* the bit cuts, not *how* it gets there. Traversal bugs (wrong corner sequence, diagonal crossings) are invisible in the overlay. If test cut shows diagonal witness marks or unexpected straight-line artifacts: inspect NC file directly, check first G1 move distance after each plunge.

---

## SPATIAL LANGUAGE

**Frames:**
- **SVG frame** -- math and code only
- **Document window** -- what Jason sees on screen. "Right corner of the document."
- **Object frame** -- derived from centerline/widget. "Top of the body."

Read context. When clear, proceed. When ambiguous, ask once. Name the frame when stating positions if any chance of confusion.

**Centerline (unlabeled):** look for line roughly parallel to feature's long axis · line features are symmetric around · long straight line not a cut path. Ask once to confirm. Multiple centerlines: resolve from context, ask once if ambiguous.

**Widget session protocol:**
1. Find `id` containing `scale-widget`
2. Read `data-origin-x`, `data-origin-y`, `data-axis-angle`
3. Read TOP/BOT/L/R labels inside box -- authoritative for object frame
4. Confirm scale against SCALE block
5. Declare at session start

*[Full widget spec: jhg_scale_widget_1_0.md](https://raw.githubusercontent.com/JasonHunt3r/jhg-shop-docs/main/jhg_scale_widget_1_0.md)*

**Profile: 17° Body Rotation**
- Body SVGs rotated 17° CCW to fit 18×18" bed
- Object frame "up" = neck end = upper-right of document window
- Widget on neck centerline, `data-axis-angle="17"`
- "Top of the body" = neck end. "Top of the document" = document window. Read context.

**Quadrants:** A=top-left · B=top-right · C=bottom-left · D=bottom-right. State at start of multi-panel sessions.

**Arrows to outside face:** tip = `curve_extreme + STANDOFF` outward. Wrong sign = points to inside face.
