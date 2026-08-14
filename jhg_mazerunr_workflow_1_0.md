# JHG MazeRunr -- Continuous Pocket Clearing
Jason Hunter Guitars -- Design Record

*MazeRunr is an unfinished subsystem, not a lost one. G-code emission was never built, so it has never produced a runnable NC file -- nothing in the current job library depends on it or is silently degraded by its absence from the doc set. Treat this as a design record to resume from, not a bug list to work through. Recovered 2026-08-13 from a 52-conversation audit of pre-Claude-Code sessions (`jhg_orphaned_knowledge_1_1.md` in `Transition Support Docs/`); source code lives at `jobs/maze-runner/ring_stitch_v4.py` and `ring_stitch_v5.py`.*

---

## What It's For

Continuous inward-winding pocket clearing -- an alternative to the standard rough/penultimate/final offset-stack approach used elsewhere in the library, for pockets where a maze-like continuous path is more efficient than repeated perimeter offsets. Shop term: **maze maker**.

---

## Current State (v6 / `_2` files)

**Ring Stitch:** two interleaved paths, **Come** and **Go**, tiling the pocket floor.

- Come winds CW, Go winds CCW
- `S_COME = PTW / n`, `S_GO = PTW / (n+1)`, `n` forced odd
- **Direction convention -- read carefully:** Come runs in *reverse* of its generated direction (the tool enters at the center-lane endpoint and spirals outward). Go runs as generated (enters at the origin, spirals inward). Every natural-language reference to "start" or "end" in this doc means **toolpath** direction (what the bit actually does), not **generator** direction (the order the points were computed in) -- these are opposite for Come and the same for Go, and mixing them up is the most likely source of a future misread.
- Cut sequence: plunge at Come's start → cut Come outward → at cutting depth, G1 traverse to Go's entry point → cut Go inward → hand off to the next module with no retract

## Gap End Condition Is Shape-Independent

A ring ends wherever `dist(current_pos, start_pos) <= s_adjusted`, regardless of pocket geometry. Don't just stop at the first resampled point that satisfies this -- interpolate between the last point outside the threshold and the first point inside it, so the endpoint lands *exactly* at `s_adjusted` rather than wherever the sampling happened to land.

## Adaptive Overlap Search (the strongest idea in the line)

Fixed `OVERLAP_RATIO = 0.4` was replaced with a search over `OVERLAP_MIN=0.10, OVERLAP_MAX=0.60, STEP=0.01`, finding the lowest odd `n` satisfying two conditions:

1. Outer lanes reach the wall: `S_COME / 2 <= BIT_R`
2. The last ring's interior is provably covered: a PyClipper inset by `BIT_R` collapses to nothing

Result: strat pickup pocket, peanut, and kidney shapes all resolved to **zero center lanes**. The same inset-collapse test replaced the old perimeter-size termination guard.

This is **coverage-as-termination** -- proving the remaining area is fully swept -- rather than a geometric approximation of when to stop. It's the one piece of this subsystem that isn't just standard offset-ring clearing.

## PTW Is a Known Shortcut, Not a Measurement -- OPEN

`POCKET_THROAT_WIDTH` currently comes from the bounding box. Reasonable for a convex shape with a clear long axis. For an L-shape with an 80×60 bounding box, the rings collapse long before tiling the full 80mm of floor -- spacing comes out too sparse.

**Principled fix, not yet built:** derive PTW from the pocket's *inner boundary* -- the widest chord perpendicular to the tiling axis -- or use Shapely's `medial_axis` to get the true pocket centerline for arbitrary shapes.

**Before reaching for Shapely here:** see the Offset Method section in `jhg_shop_file_standards_1_9.md` -- `Polygon.buffer()` clips peninsula features, and pulling Shapely back into geometry work for `medial_axis` re-arms that failure mode for anything else it touches in the same session. `medial_axis` itself may not have this problem, but confirm before assuming it's safe by association.

**Open strategic question, never resolved:** how much of ring-offset-plus-inset-collapse clearing is just reinventing standard CAM contour clearing. Honest answer from the session that raised it: partially, yes -- the offset-ring and inset-collapse geometry is standard. The adaptive minimum-ring-count search with *guaranteed* coverage is not, and is the part worth keeping if this gets rebuilt on top of a standard library instead of by hand.

## Strategy Library

Shape-specific solutions, selected explicitly rather than inferred:

- **Strategy A** -- base ring stitch (strat pickup ring pocket)
- **Strategy B** -- per-pass ring extent recomputed at each lane's tile position, with axis detection that flips for wide corridors (peanut / two overlapping circles)
- **Strategy C** -- curved centerline for crescent shapes, where straight passes would cut into the workpiece (Rockin Kidney). **Written, never proven, currently commented out.**

## Failed Approaches -- Do Not Revisit

Eight walker variants were built and rejected: step-and-check flow logic, contour-following with ring close, no-close ring with nearest-point join, distance-to-next-lane trigger, perpendicular-distance trigger, local-minimum-distance trigger, Shapely arc-length parameterization, segment-by-segment cut-and-reconnect. All produced axis-aligned rectangles, concentric rings with diagonal stitches, or chaotic early transitions.

**Root cause, and the reason none of the eight worked:** this is a **traversal** problem, not a geometry problem. Every walker variant was a fix aimed at geometry when the actual defect was in how the path decided where to go next. If this gets picked back up, start from that framing rather than from another walker variant.

Also discarded as failed-idea artifacts, not worth re-trying: seam-endpoint park logic, boustrophedon (back-and-forth raster) alternation.

---

## If Resuming This

G-code emission (the step that turns Come/Go point lists into an actual NC section) was never built. That's the next real step, not a further geometry refinement -- the geometry side (Ring Stitch v6, adaptive overlap search) is further along than the emission side. Before writing the emitter, read `ARC-FIT PATH ROTATION` and `GO-AND-COME ARC REVERSAL` in `jhg_gcode_hygiene_1_9.md` -- MazeRunr's Come/Go handoff-with-no-retract is structurally similar to a go-and-come pair and is exactly the kind of place the plunge-index bug class could reappear if the point lists aren't rotated to match wherever the toolpath actually enters.
