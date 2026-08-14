# JHG Orphaned Knowledge — Extraction 1.1

**Revision note (1.0 → 1.1):** reassessed against
`jhg_chronology_and_provenance_1_0.md`. Seven items closed, four downgraded,
one elevated. Version 1.0 was written without chronology and treated several
already-solved problems as open.

Source: audit of 52 conversations in the ClaudeCAM Claude Project, cross-checked
against repo state 2026-03-28.

---

## VERDICT SUMMARY

| Item | 1.0 verdict | 1.1 verdict |
|---|---|---|
| A. Uncommitted docs | Orphaned | **Reported handled** — public GitHub disagrees |
| B1. `buffer()` vs `offset_curve()` | Active rule | **Archaeology** — re-arms if Shapely returns |
| B2. MITER_LIMIT | Undocumented risk | **Downgraded** — no longer touches finish path |
| B3. Pocket corner pre-fillet | Unverified | Unverified — unchanged |
| C. POCKET_EXPAND | Orphaned | **In generator comments** — stale at 0.45 |
| D1. C1/C2 bridging | Bug class | **ELEVATED** — got past the verifier |
| D2. Reversed-list index | Unverified | Partly covered by hygiene 1.7 |
| D3. Arc fitter / concave | Active | Active — unchanged, from latest session |
| D4. Near-duplicate dedup | Orphaned | Covered in runbook + workflow |
| D5. Arc radius / error 33 | Partial | **CLOSED** — fully documented |
| E. MazeRunr | Orphaned | **Reframed** — unfinished subsystem, not lost |
| F. Machine envelope | Orphaned | Mostly covered; Z-budget math may not be |
| G1. Horn-tip machine halt | Oldest open bug | **CLOSED** — avoided by no-hand-patch rule |
| G2. Oblong holes | Open | **CLOSED** — helical arcs + HOLE_OFFSET |
| G3. ARC_MAX_R | Root cause unknown | **Probably explained** — undersampling |
| G4. Body outline winding | Unresolved | **CLOSED** — fixed and documented |
| H. Vocabulary | Orphaned | Unchanged |
| I. Verification tools | Orphaned | **CLOSED** — already in runbook |

---

# CLOSED

Retained as one-liners for trace-back. No action needed.

**G1 — horn-tip machine halt.** Caused by hand-patching NC with G1 substitutions
for arcs. Standards 1.8: *"change it in the `.py` and regenerate — do not patch
output files directly."* The failure mode is structurally unreachable now. The
session's own choice to patch NC rather than fix the generator is what the rule
was written against. Unrecorded isolation-test result no longer matters.

**G4 — body outline winding.** Fixed in *Panel A revision*: body path reversed
to CW/conventional after Y-flip, with the resulting `offset_open_curve_outward`
→ `inward` error caught in the same session. Documented in hygiene 1.7 lines
63–66, tagged CONFIRMED March 2026. The later "unresolved" note was a regression
against an existing written rule, not a knowledge gap.

**G2 — oblong / undersize holes.** Two fixes landed independently: Neo 3 moved
holes to helical G2/G3 ramps + finish orbits + spring passes (native arcs
replacing linearized circles — addresses out-of-roundness); *SVG design file
handoff* added `HOLE_OFFSET = 0.5mm` to cut 11.0mm real from 10.0mm design
(parameterizes the delta instead of chasing it). **Residual:** caliper the bit
to settle 6mm vs 6.35mm. One measurement, not an investigation.

**D5 — arc radius consistency / GRBL error 33.** Hygiene 1.7 has a full
`GO-AND-COME ARC REVERSAL` section with the five-step I/J recomputation
algorithm and an explicit "never flatten arcs to G1."

**D4 — near-duplicate point dedup.** 0.05mm NC-space threshold appears in
runbook and workflow.

**I — verification tools.** NCViewer / CAMotics / Fusion limitation already in
runbook and workflow.

**F — machine envelope, mostly.** 500W and TTC450 in hygiene; SAFE_Z / PARK_Z in
hygiene and standards. **Possible gap:** the Z-budget arithmetic — 3/4" spoilboard
+ 5/8" stock consumes ~35mm, leaving ~45mm usable from work Z=0; park at Z=40;
Z=300 for bit-change clearance. Confirm that reasoning is written somewhere and
not just the resulting constants.

---

# DOWNGRADED

## B1. `buffer()` clips peninsulas — archaeology, with a trigger

`Polygon.buffer()` treats the shape as a filled area, reads peninsulas (horns,
headstock tips) as concavities, and replaces pointed tips with flat chords.
`offset_curve(LinearRing(...))` offsets along local normals without topological
clipping.

**Why this is no longer an active rule:** Shapely offsetting left the pipeline.
Current stack is PyClipper for rough/penult offsets and Illustrator bezier
offset + Bernstein correction for the finish path.

**Trigger that re-arms it:** *Polishing MazeRunr* proposes reintroducing Shapely
for `medial_axis` to fix PTW. If Shapely comes back into geometry work, this
becomes live again. Keep the finding attached to that decision.

Value retained: this is the origin of "loopty" and the reason the pipeline
looks the way it does.

## B2. MITER_LIMIT — write the value down, drop the alarm

`MITER_LIMIT = 20`, `JT_MITER` not `JT_ROUND`. Origin: a ~6.35° turn at the
lower-left horn needed a ~57mm spike; `MITER_LIMIT = 10` capped it at 31.75mm
and truncated the corner by ~25mm.

**Severity downgrade:** from Phase 4 onward the finish path is the verified
Illustrator bezier read 1:1, so PyClipper only shapes passes that leave stock.
Miter truncation can no longer reach finished geometry. Document the value and
the reason; stop treating it as a part-accuracy hazard.

## C. POCKET_EXPAND — not orphaned, just stale

*Panel A tighten up* wrote the direction convention and value history into the
generator's code comments — where anyone changing it would look. Absent from the
markdown library, present at the point of use.

- **larger value → looser fit** (negated at call site:
  `offset_open_curve_inward(pocket_pts_mm, -POCKET_EXPAND)`)
- XY is `.3f`, Z is `.1f`

**Only real gap:** the in-code history stops at 0.45. Full ladder is
0.40 (tight) → 0.50 (loose) → 0.45 → 0.43 → 0.42 → **0.41 current**. Refresh the
comment block.

## D2. Reversed-list index arithmetic — partly covered

Hygiene 1.7 line 66 covers the general case (`list(reversed(verts))` after
offsetting for interior pockets). The specific failure — calling side-ranking
functions on an already-reversed list — is not stated. The arithmetic form:

```
reverse_side_idx = (n - 2 - forward_side_idx) % n
```

**[VERIFY]** against the generator before promoting.

## G3. ARC_MAX_R — probably explained, not mysterious

A 170mm-radius arc produced a 393mm endpoint mismatch; `ARC_MAX_R` was dropped
200 → 100 as a working fix, then restored to 200 in *Troubleshooting cut file*
with `samples_per_curve` raised 30 → 60 instead.

**Reassessment:** the restored value plus the sampling fix strongly suggests the
degenerate fit was the same undersampling problem as D3 — the fitter spanning
too much geometry per arc — rather than a separate defect. Downgrade from "root
cause never found" to "plausibly undersampling; confirm."

Still deliberately not a rule: the 5px close-point cleanup threshold.

---

# ACTIVE

## D1. C1/C2 bridging — ELEVATED

After each body come-leg, traversing to `pocket_finish_mm[0]` (C1) instead of
`pocket_finish_mm[-1]` (C2) drives a ~57mm diagonal straight across the neck
pocket. Found in penult, finish, and spring sections after having been fixed
once in the rough pass.

**Why this is now the top item:**

1. **It got past the verifier.** Chronology: the 33-check verifier was built in
   *Cut file issues in updated version*; the bridging bug was found in
   *Panel A revision*, after. The verifier did not catch it.
2. **The diagnostic already exists but is not automated.** Runbook line 225:
   *"Traversal bugs (wrong corner sequence, diagonal crossings) are invisible in
   the overlay... inspect NC file directly, check first G1 move distance after
   each plunge."* That is precisely this bug's signature.
3. **Exposure is narrower than feared.** Panel C uses a single closed body loop
   with no C1/C2 stitching and is structurally immune. This is Panel-A-only.

**Action:** add a check to the verifier asserting the first G1 move distance
after each plunge is below a threshold. Turns a written diagnostic into an
automated gate, and covers the one section family that has produced repeat
failures.

## D3. Arc fitter spans concave features

At a concave neck-joint dip the fitter produced a large-radius G2 (r=17.5mm,
center inside the body) instead of hugging the wall — the loopty-loo that arcs
the bit into the part. Rough and penult were unaffected because polygon-based
offsetting fed the fitter differently-spaced samples.

Fix: `samples_per_curve` 30 → 60.

**Still active.** From the most recent session, not superseded, and
`samples_per_curve` appears in no repo doc. Related to G3 — likely the same
underlying cause.

## B3. Pocket corner pre-fillet

Fillet pocket corner tips at `r = BIT_R + 0.2mm` **before** PyClipper offsetting,
so miter spikes never enter the toolpath.

**[VERIFY]** — from Phase 2; unclear whether it survived the move to the
two-module C1/C2 architecture.

---

# E. MazeRunr — reframed

Absent from the repo, but this is **an unfinished subsystem, not lost knowledge
affecting current cuts.** G-code emission (Step 8) was never built; MazeRunr has
never produced a runnable NC file. Nothing here is silently degrading output.

Treat as a design record to resume from, not a documentation gap to patch.

**Current state (v6 / `_2` files):**

- COME winds CW, GO winds CCW; `S_COME = PTW/n`, `S_GO = PTW/(n+1)`, `n` odd
- COME runs *reverse* of generated direction (enters at center-lane endpoint,
  spirals outward); GO runs as generated. All natural-language start/end refers
  to **toolpath** direction.
- Sequence: plunge at COME start → COME outward → at-depth G1 traverse to GO
  entry → GO inward → hand off with no retract
- Gap end condition is shape-independent: ring ends where
  `dist(current, start) <= s_adjusted`, interpolated to land exactly there

**E3 — adaptive overlap search** (the strongest idea in the line). Fixed
`OVERLAP_RATIO = 0.4` replaced by a search (MIN 0.10, MAX 0.60, STEP 0.01) for
the lowest odd `n` satisfying: outer lanes reach the wall (`S_COME/2 <= BIT_R`)
**and** the last ring's interior provably covered (PyClipper inset by `BIT_R`
collapses). Same inset-collapse test replaced the perimeter-size termination
guard. Result: strat, peanut, and kidney all resolved to zero center lanes.
Coverage-as-termination rather than geometric approximation.

**E4 — PTW is a shortcut. OPEN.** Currently from bounding box. For an L-shape
with an 80×60 bbox the rings collapse long before tiling 80mm, so spacing comes
out too sparse. Principled fix: widest chord of the *inner boundary*
perpendicular to the tiling axis, or Shapely `medial_axis`. **See B1 before
reintroducing Shapely.**

**E5 — Strategies.** A: base ring stitch. B: per-pass ring extent at each lane's
tile position, axis detection flips for wide corridors. C: curved centerline for
crescents — written, unproven, commented out.

**E6 — Do not revisit.** Eight walker variants failed: step-and-check flow,
contour-follow with ring close, no-close ring with nearest-point join,
distance-to-next-lane trigger, perpendicular distance trigger, local minimum
distance trigger, Shapely arc-length parameterization, segment-by-segment
cut-and-reconnect. Root cause: traversal problem, not geometry problem. Also
removed as failed-idea artifacts: seam-endpoint park logic, boustrophedon
alternation.

---

# A. Document sync

Reported handled in the Claude Code project. Public GitHub `main` as of this
writing still shows the Mar 28 commit — runbook `1_2`, hygiene `1_7`, and 404s
on `1_3`, `1_8`, `mazerunr_workflow_2`, `library_index`, `session_starter`. No
other branches.

If Claude Code is now the source of truth, this is a non-issue and the repo is a
historical snapshot. Worth stating explicitly somewhere, since the runbook
instructs Claude to fetch docs from those GitHub raw URLs at session start — an
instruction that currently serves stale versions.

---

# H. Vocabulary

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
| Big Ears | Panel C after EMG ear pockets widened 3.26mm/end |

---

# WHAT TO ACTUALLY DO

1. **Add the first-G1-move-after-plunge check to the verifier.** Highest value —
   automates a diagnostic already written in the runbook, covering the one bug
   class known to have slipped past the existing 33 checks.
2. **Caliper the bit.** Closes G2's residual outright.
3. **Refresh the POCKET_EXPAND comment block** from 0.45 to the full ladder
   ending at 0.41.
4. **Document `samples_per_curve` and MITER_LIMIT values** with their reasons.
5. **Confirm B3 and D2** against the current generator before promoting either.
6. **State the GitHub-vs-Claude-Code source-of-truth decision** so the runbook's
   fetch instruction isn't pointing at stale docs.
