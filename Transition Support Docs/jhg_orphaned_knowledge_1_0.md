# JHG Orphaned Knowledge — Extraction 1.0

Knowledge that exists **only in Claude project conversations** and is absent from
`github.com/JasonHunt3r/jhg-shop-docs` as of repo state 2026-03-28.

Source: audit of 52 conversations in the ClaudeCAM Claude Project.
Confidence is marked per item. Items marked **[VERIFY]** are drawn from
conversation summaries rather than from a read of the working code — confirm
against the generator before promoting to a rule.

---

## A. Documents produced but never committed

| Document | Produced in | Contains |
|---|---|---|
| `jhg_claudecam_runbook_1_3.md` | *Cut file issues in updated version* | NC Delivery Gate |
| `jhg_gcode_hygiene_1_8.md` | *Cut file issues in updated version* | NC Output Verification section |
| `jhg_mazerunr_workflow_2.md` | *I/O error during file export* | Entire MazeRunr subsystem, v2 state |
| `jhg_library_index.md` | *Document insertion suggestions* | Doc table + raw URLs (currently 404) |
| `jhg_session_starter.md` | *Document insertion suggestions* | Standard + library-update session templates |
| `jhg_pocket_clearing_algorithm_1.md` | *Pocket Clearing Algorithm* | Formal clearing spec |
| 5–6 doc inserts | *Panel C workfiles analysis* | Drafted for hygiene / standards / workflow / troubleshooting |

**Action:** confirm which of these exist locally, then commit. The repo being
stale directly undermines the "fetch proven files at session start" strategy
adopted specifically to stop bug reintroduction.

---

## B. Geometry engine — undocumented root causes

### B1. Offset method: `buffer()` clips peninsulas

`Polygon.buffer()` treats the shape as a filled area. Peninsula features —
guitar horns, headstock tips — are read as concavities and get clipped, with
pointed tips replaced by flat chords.

Use `offset_curve(LinearRing(...))` instead. It offsets along local normals
without topological clipping.

Secondary effect: the implicit Z-close segment in an SVG path creates a
junction artifact at the closure. Fix by explicitly sampling the closing edge
as discrete points rather than relying on LinearRing's implicit closure.

Also relocate the toolpath start off the horn tip — entry marks are visible
there. Middle of a flat edge is the correct home.

*Source: Converting SVG to 3D CNC cut files. High confidence — this was the
breakthrough after averaged-vertex-normal and miter-spike approaches failed.*

### B2. MITER_LIMIT is load-bearing and undocumented

A shallow turn angle needs a long miter spike. At the lower-left horn
(~6.35° turn) the required spike was ~57mm; `MITER_LIMIT=10` capped it at
31.75mm and truncated the corner by ~25mm.

- `MITER_LIMIT = 20` preserves horn tips without runaway spikes elsewhere.
- Use `JT_MITER`, not `JT_ROUND`, to preserve sharp concave corners at neck
  pocket junctions.

The repo mentions "miter" once per doc but never states the value or why.

*Source: Reviewing the hand off; Converting SVG to G-code cut files. High confidence.*

### B3. Pocket corner pre-fillet

Fillet pocket corner tips at `r = BIT_R + 0.2mm` **before** PyClipper offsetting
to prevent miter spikes appearing in the toolpath rather than the geometry.

*Source: Adjusting cut file for body fit. [VERIFY] against generator.*

---

## C. POCKET_EXPAND — zero occurrences in the repo

The single most-tuned parameter in the system has no documentation.

**Sign convention is counterintuitive.** Because the value is negated at the
call site — `offset_open_curve_inward(pocket_pts_mm, -POCKET_EXPAND)` — the
natural-language direction inverts:

- **larger** POCKET_EXPAND → **looser** neck pocket fit
- **smaller** POCKET_EXPAND → **tighter** fit

**Value history (physical test cuts, MDF):**

| Value | Result |
|---|---|
| 0.40 | too tight |
| 0.50 | too loose |
| 0.45 | interim |
| 0.43 | cut on Panel 3 |
| 0.42 | tightening |
| 0.41 | current at last session |

**Format constraint:** XY is `.3f`, Z is `.1f`. A 0.45-class value is safe for
XY math but would not survive Z formatting.

*Source: Panel A tighten up; Panel A revision. High confidence.*

---

## D. Bug classes worth a standing check

### D1. C1/C2 bridging — the diagonal crosscut

After each body come-leg, traversing to `pocket_finish_mm[0]` (C1) instead of
`pocket_finish_mm[-1]` (C2) produces a ~57mm diagonal straight across the neck
pocket. Appeared in penult, finish, and spring sections.

This class recurred: it was fixed once in the rough pass, then found again
later in three other sections. **When this bug is found anywhere, audit every
section in the file before declaring it fixed.**

*Source: Panel A revision. High confidence — cost real cuts.*

### D2. Reversed-list index arithmetic

Calling side-ranking functions on a reversed vertex list produces diagonal
crossings on return passes. Compute the index arithmetically instead:

```
reverse_side_idx = (n - 2 - forward_side_idx) % n
```

*Source: Panel C workfiles analysis. [VERIFY].*

### D3. Arc fitter spans concave features

At a concave neck-joint dip the arc fitter produced a large-radius G2 arc
(r=17.5mm, center inside the body) instead of hugging the wall — the
"loopty-loo" that arcs the bit into the part.

Rough and penult passes were unaffected because polygon-based offsetting fed
the fitter differently-spaced samples.

Fix: raise `samples_per_curve` 30 → 60 so the fitter cannot span the dip in a
single arc.

*Source: Troubleshooting cut file with jhg collaboration guidelines. High confidence.*

### D4. Near-duplicate points create micro-jogs

Points ~0.001mm apart at bezier segment boundaries produce 90° micro-jogs and a
visible bump. Deduplicate **in NC space after coordinate conversion**, at a
0.05mm threshold.

*Source: SVG design file handoff and review. High confidence.*

### D5. Arc radius consistency (GRBL error 33)

When a start point changes, I/J must be recomputed. Fitted arcs need endpoint
projection and actual-position tracking through the emitter, or start-to-center
and end-to-center distances disagree and GRBL throws error 33.

*Partially covered in repo (error 33 appears in three docs) — the I/J
recomputation-on-start-change rule is the part worth confirming is explicit.*

---

## E. MazeRunr — entirely absent from the repo

Zero occurrences of "maze", "MazeRunr", "PTW", "throat", "pyclipper", or
"shapely" across all eight repo docs.

### E1. Ring Stitch, current state (v6)

Two interleaved paths, COME and GO, tiling the pocket floor.

- COME winds CW, GO winds CCW
- `S_COME = PTW / n`, `S_GO = PTW / (n+1)`, `n` forced odd
- **Direction convention:** COME runs in *reverse* of generated direction (tool
  enters at the center-lane endpoint and spirals outward). GO runs as generated
  (enters at origin, spirals inward). All natural-language references to
  start/end mean **toolpath** direction, not generator direction.
- Cut sequence: plunge at COME start → cut COME outward → at-depth G1 traverse
  to GO entry → cut GO inward → hand off to next module with no retract

### E2. Gap end condition is shape-independent

A ring ends wherever `dist(current_pos, start_pos) <= s_adjusted`, regardless of
pocket geometry. Interpolate between the last point outside and first point
inside so the endpoint lands *exactly* at `s_adjusted` — don't just stop at the
first qualifying resampled point.

### E3. Adaptive overlap search replaced fixed OVERLAP_RATIO

Fixed `OVERLAP_RATIO = 0.4` was replaced with a search
(`OVERLAP_MIN=0.10`, `MAX=0.60`, `STEP=0.01`) finding the lowest odd `n`
satisfying two conditions:

1. outer lanes reach the wall: `S_COME/2 <= BIT_R`
2. the last ring's interior is provably covered: PyClipper inset by `BIT_R`
   collapses

Result: strat pocket, peanut, and kidney all resolved to **zero center lanes**.
The same inset-collapse test replaced the old perimeter-size termination guard.

This is coverage-as-termination rather than geometric approximation, and it is
the strongest idea in the MazeRunr line.

### E4. PTW is a known shortcut, not a measurement — OPEN

`POCKET_THROAT_WIDTH` currently comes from the bounding box. For a convex shape
with a clear long axis that's reasonable. For an L-shape with an 80×60 bbox,
the rings collapse long before tiling 80mm of floor — spacing comes out too
sparse.

Principled fix: derive PTW from the **inner boundary** — widest chord
perpendicular to the tiling axis. Or use Shapely's `medial_axis` to get the
true pocket centerline for arbitrary shapes.

**Open strategic question, unresolved:** how much of ring-offset + inset-collapse
is reinventing standard CAM contour clearing. Honest answer from that session:
partially yes. The offset-ring and inset-collapse geometry is standard;
the adaptive minimum-ring-count-with-guaranteed-coverage search is not.
Shapely could likely replace the hand-rolled PTW math entirely.

### E5. Strategy library

Shape-specific solutions are named strategies with explicit selection logic:

- **A** — base ring stitch (strat pickup ring pocket)
- **B** — per-pass ring extent at each lane's tile position, axis detection
  flips for wide corridors (peanut / two overlapping circles)
- **C** — curved centerline for crescent shapes where straight passes cut into
  the workpiece (Rockin Kidney). **Written, not proven, commented out.**

### E6. Failed approaches — do not revisit

Eight walker variants were built and rejected: step-and-check flow logic,
contour-following with ring close, no-close ring with nearest-point join,
distance-to-next-lane trigger, perpendicular distance trigger, local minimum
distance trigger, Shapely arc-length parameterization, segment-by-segment
cut-and-reconnect. All produced axis-aligned rectangles, concentric rings with
diagonal stitches, or chaotic early transitions.

Root cause: this is a **traversal** problem, not a geometry problem. Every
walker fix was a bandaid.

Also removed as failed-idea artifacts: seam-endpoint park logic, boustrophedon
alternation.

*Source: Continuous maze pocket clearing algorithm; Algorithm for shape
clearing; Maze Runner development progress; I/O error during file export;
Polishing MazeRunr.*

---

## F. Machine envelope facts

- **TwoTrees TTC450 PRO**, 460×460mm bed, **500W** spindle, ER11 collets.
  (An earlier log recorded 1000W — that was wrong and was corrected.)
- MKS DLC32 V2.1, GRBL, factory-default settings, files from SD card.
- **Z budget:** 3/4" MDF spoilboard + 5/8" work stock consumes ~35mm, leaving
  ~45mm usable from work Z=0. Park at **Z=40mm**; Z=300mm is the manual-raise
  target for bit insertion clearance.
- Machine is not rigid enough for full body cutting. Production flow is
  bandsaw rough → router table with templates for final shaping → CNC for
  precision work only (neck pockets, pickup routes).

*Source: CNC router bed size compatibility; Converting SVG to 3D CNC cut files.*

---

## G. Open / unresolved

### G1. Machine halt at horn tip — NO ROOT CAUSE

Patched NC files halted the machine at the horn tip with the bit at cutting
depth, XY stopped permanently, spindle running. **Progress bar showed 0%
throughout execution — even after holes completed successfully.** That 0% is
the strongest unexplained clue.

Ruled out: filename length, arc radius mismatches, file encoding, SD card.

A minimal single-replacement test file (`jhg_hs_neo3_test1.nc`, first rough-pass
G3 pair only) was built to isolate whether G1 substitution for arcs is
fundamentally incompatible with this controller. **Result never recorded.**

*Source: CNC headstock arc bump troubleshooting. This is the oldest open bug.*

### G2. Oblong tuner holes

Programmed 11.2mm cut oversize; stepped 11.2 → 11.1 → 11.0. Working theory is
tool flex under lateral load. Secondary hypothesis, never confirmed: **the bit
may be 6mm, not 6.35mm.** Deferred for caliper measurement — still deferred.

Three separate chats opened on this and all three died before reading the file.

### G3. ARC_MAX_R degenerate fits

A 170mm-radius arc produced a 393mm endpoint mismatch. `ARC_MAX_R` was lowered
100 ← 200 as a working fix, later restored to 200 to match shop standard, with
`samples_per_curve` raised instead. Root cause never established.

Related: the 5px close-point cleanup threshold was never confirmed as a
principled general solution. Both were explicitly *not* promoted to rules.

### G4. Body outline winding on the two-module path

The SVG path direction (C1 → perimeter → C2) produces the wrong winding in NC
space for an outside profile, cutting climb instead of conventional. Fix
requires reversing path direction without touching the offset functions.
Status at last session: unresolved.

---

## H. Vocabulary worth preserving

Both registers, deliberately — shop language for talking, engineering language
for searching.

| Shop | Engineering |
|---|---|
| loopty / loopty-loo | offset curve self-intersection; arc direction flip in finish pass |
| warble | G2 curvature discontinuity from bezier handle-length mismatch |
| bump / divot | offset artifact at degenerate bezier junction (`ctrl2 == end`) |
| go-and-come | bidirectional finish pass pair |
| spring pass | low-load cleanup lap at final offset |
| onion skin | material intentionally left under the part |
| maze maker | continuous inward-winding pocket clearing |
| Come / Go | the two interleaved spiral phases |

---

## I. Verification tools (off-pipeline)

Fusion 360 does not meaningfully import externally-generated NC for editing or
simulation. For checking ClaudeCAM output before it reaches the machine:

- **CAMotics** — material removal simulation
- **NCViewer** — fast browser-based check
- **CNCjs**, **UGS** — GRBL-dialect senders with visualization

*Source: Importing NC files to Fusion 360 CAM.*
