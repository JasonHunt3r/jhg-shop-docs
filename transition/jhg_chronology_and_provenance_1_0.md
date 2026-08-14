# JHG ClaudeCAM — Reconstructed Chronology & Provenance Index 1.0

The Claude Project move on 2026-08-14 overwrote every conversation timestamp.
The 44-chat bulk batch was re-sorted into UUID-lexicographic order in pages of
20, which carries zero chronological information. This document reconstructs
the real order from evidence internal to the work.

**Purpose:** trace-back. When a current issue surfaces, find where it first
appeared and what shipped downstream of it.

## Confidence tiers

| Tier | Basis |
|---|---|
| **[A] Anchored** | Dated filename or transcript path inside the session |
| **[L] Ladder** | Fixed by a monotonic doc-version or parameter chain |
| **[M] Move-order** | Preserved move sequence of the 6-chat batch (reverse chronological) |
| **[I] Inferred** | Content adjacency only — treat as approximate |

## Hard anchors

| Date | Session | Evidence |
|---|---|---|
| 2026-03-08 | Fender Strat neck pocket SVG design | `/mnt/transcripts/2026-03-08-00-28-37-strat-neck-pocket-svg-cnc.txt` |
| 2026-03-11 09:37 | 4 Panel take 1 (old) | `jhg_body_audit_1_1_backup_20260311_093717.svg` |
| 2026-03-11 12:47 | 4 Panel Body Layout | `/mnt/transcripts/2026-03-11-12-47-00-jhg-body-audit-svg-editing.txt` |
| 2026-03-21 | Body cut file review and clarification | `jhg_session_handoff_20260321_panelC.md` |
| ~2026-03-27/28 | Document insertion suggestions | Repo first commit dated Mar 28 01:44 |
| 2026-08-05 | Both migration sessions | Timestamps survived (already in project) |

## Version ladders used as spine

```
standards       1 → 1.2 → 1.2.1 → 1.3 → 1.4 → 1.5 → 1.6 → 1.8
hygiene             1.1 → 1.2 → 1.3 → 1.4 → 1.5 → [1.6?] → 1.7 → 1.8
troubleshooting     1.1 → 1.2 → 1.3 → 1.4
svg workflow        1.1 → 1.2 → 1.3 → merged into ClaudeCAM 1.0
runbook                                        1.2 → 1.3
POCKET_EXPAND   0.4 → 0.5 → 0.45 → 0.43 → 0.42 → 0.41
ROUGH_LEAVE     0.5 → 1.0 → 2.0
PENULT_LEAVE    0.2 → 0.3 → 0.5
ring stitch     v1 → v2 → v3 → v4 → v5 → v6 → _2 files
headstock       tabs → Neo 2 → Neo 3 → Neo 4
Big Ears                        _4.zip → _5.zip
```

`hygiene 1.6` is unaccounted for. Best guess: it absorbed the five lessons
drafted in *Panel C workfiles analysis*.

---

# PHASE 0 — Pre-library headstock (February)

Before any shop docs existed. This is where the vocabulary and the core
geometry findings were born.

| # | Session | Marker | Tier |
|---|---|---|---|
| 1 | Converting SVG to 3D CNC cut files | loopties; `buffer()` → `offset_curve(LinearRing())`; **nugget library concept originates here** | I |
| 2 | File editing assistance | horn truncation + divot + non-circular circles; script never existed | I |
| 3 | Tuner holes cutting as oblongs | `headstock_MDF_fullcut_tabs_2.nc` | I |
| 4 | Slowing circle cut path to fix oblongs | same file | I |
| 5 | CNC circular hole cutting issues | `..._slow_holes.nc` = output of #3/#4 | L |
| 6 | G code file upload issues | upload mechanics | I |
| 7 | CNC coordinate centering and tool direction | right long side bezier; offset direction inversion; holes 9.5/10.0mm | I |
| 8 | CNC toolpath generation from SVG with bit compensation | left long arc S4 warble; references "a previous broad correction attempt" (= #7) | I |

**Note:** the `MDF_fullcut_tabs` filename convention appears only in this phase.
It is a reliable early-era marker.

---

# PHASE 1 — Shop docs born, Neo 2 → Neo 3 (late Feb – early Mar)

| # | Session | Marker | Tier |
|---|---|---|---|
| 9 | Shop File Standards origin | updates `jhg_shop_file_standards_1.md`; SVG scale rules; 2.8346 px/mm recovered from 55.712mm physical measurement | L |
| 10 | Reviewing the hand off | Neo 2; hole 11.2 → 11.1 → 11.0; **MITER_LIMIT 10 → 20** | L |
| 11 | Headstock Neo 3 work | **G1/G2/G3 hybrid established**; GRBL error 33; creates standards 1_2, hygiene 1_1, workflow 1_1, troubleshooting 1_1 | L |
| 12 | CNC headstock arc bump troubleshooting | cites standards `1_2_1`; **machine halt at horn tip — never resolved** | L |

---

# PHASE 2 — Strat pocket geometry + body panels (Mar 8 – 21)

| # | Session | Marker | Tier |
|---|---|---|---|
| 13 | Fender Strat neck pocket SVG design | **Mar 8**; fillet tangency method; 7-item postmortem | A |
| 14 | Strat Pocket Start | `_shape_1` → `_shape_2` | L |
| 15 | Strat Pocket Struggles | `_shape_2_4`; inscribed fillet circle r=25px | L |
| 16 | Converting SVG to G-code cut files | 4-panel audit `_1_1` built; JT_MITER + MITER_LIMIT 20; four rot17 file sets | I |
| 17 | 4 Panel take 1 (old) | **Mar 11 09:37**; scale widget + panel blurbs | A |
| 18 | 4 Panel Body Layout | **Mar 11 12:47**; `_1_1_18`; widget anchor derivation | A |
| 19 | CNC router bed size compatibility | Z-travel budget; park Z=40; **500W spindle correction**; fit-test workflow designed | I |
| 20 | Neck pocket test fitting preparation | `pocket_test_offset_0_0mm_5.svg`; executes #19's workflow | L |
| 21 | Adjusting cut file for body fit | POCKET 0.4mm era; loopty trimming; `v5_rot17` | L |
| 22 | Refining cut file adjustments | G1/G2/G3 ported to body; Illustrator compat findings; workflow → 1_2 | L |
| 23 | Adjusting neck pocket bezier curves | **Bernstein CP correction originates here**; C1/C2 two-module architecture; standards 1.2.1 → 1.3, workflow 1.1 → 1.2, troubleshooting 1.1 → 1.2 | L |
| 24 | Body cut file review and clarification | **Mar 21**; Panel C 1.277mm coordinate drift recovery | A |
| 25 | Building panel C handoff | stale handoff; 2 substantive turns | I |

**Ordering caution:** #22 and #23 both claim the workflow 1.1 → 1.2 bump. They
are adjacent or concurrent; relative order is uncertain.

---

# PHASE 3 — Maze walker era (March, running parallel to Phase 2)

Distinguishable by `SVG_SCALE = 6.812 px/mm` (vs body's 2.8346) and by the
**absence** of the term "ClaudeCAM" — which was coined at Mar 28.

| # | Session | Marker | Tier |
|---|---|---|---|
| 26 | Pocket Clearing Algorithm | ROUGH_OFFSET boundary contract; algorithm md written | I |
| 27 | Continuous maze pocket clearing algorithm | ChatGPT sketch reworked; 8 approaches fail; handoff → Option C | L |
| 28 | Continuing the project | v2 walker confirmed as foundation; `maze_spiral_handoff_1_1.md` | L |
| 29 | CNC gantry not responding to file | `jhg_strat_outline_v2.svg`; cites `/mnt/transcripts/journal.txt` | I |
| 30 | Algorithm for shape clearing | walker v7 → **pivot to ring stitch v1–v3** | L |
| 31 | Maze Walker project handoff review | v3 → v4 (Come winding CW) | L |
| 32 | Maze walker idiot claude session | v4 → v5 (gap interpolation to exact `s_adjusted`) | L |

---

# PHASE 4 — Standards consolidation (Mar 21 – 28)

| # | Session | Marker | Tier |
|---|---|---|---|
| 33 | SVG cut files with bezier offset method | standards → 1.4, hygiene → 1.2; ROUGH_LEAVE 0.5 → 1.0; EMG_OFFSET introduced; **ARC_MAX_R degenerate fit found** | L |
| 34 | SVG design file handoff and review | standards 1.4 → 1.5, hygiene 1.2 → 1.3; Neo 4; NC-space dedup at 0.05mm; FEED_SPRING 669 / FEED_HELIX 1200 | L |
| 35 | CNC cutting feedback and refinements | standards → 1.6, hygiene → 1.4, workflow → 1.3; **`reverse_commands()` written**; 8-issue test-cut debrief | L |
| 36 | Examining Panel C and updating standards | hygiene → 1.5; ROUGH_LEAVE 1.0 → 2.0, PENULT 0.3 → 0.5, DIP_DEPTH 1.0 → 1.25 | L |
| 37 | Panel C workfiles analysis and handoff verification | two ruined cuts; audit-the-handoff protocol; 5 lessons drafted (probably → hygiene 1.6) | I |
| 38 | Document insertion suggestions | **~Mar 27–28. ClaudeCAM naming coined. Runbook 1_2 created. Library published to GitHub.** hygiene → 1.7, standards → 1.8, troubleshooting → 1.4 | A |

**#38 is the single most useful divider in the whole history.** Any session that
mentions "ClaudeCAM," a runbook, or fetching docs from GitHub is after it.

---

# PHASE 5 — Post-GitHub (Mar 28 → April)

All six chats in the 01:05–01:06 move batch land here.

| # | Session | Marker | Tier |
|---|---|---|---|
| 39 | Maze Runner development progress | Ring Stitch v6; three shapes; Strategies A/B/C; ends in outputs I/O error | L |
| 40 | I/O error during file export | recovers from #39's I/O error; **adaptive overlap search replaces fixed OVERLAP_RATIO**; workflow → v2 | L |
| 41 | Polishing MazeRunr | `_2` files; PTW-from-bbox critiqued; shapely `medial_axis`; "are we reinventing the wheel" — **latest MazeRunr session** | L |
| 42 | Panel A tighten up | POCKET_EXPAND 0.5 → 0.45; sign convention documented in code comments | L |
| 43 | Cut file issues in updated version | POCKET 0.45 → 0.43; **33-check verifier built**; runbook → 1_3, hygiene → 1_8 | L, M |
| 44 | Panel C ears adjustment | `pnlC_rot17_1_6_5.svg`; ears widened 3.26mm/end — **this is what makes them "Big Ears"** | L, M |
| 45 | Panel C Big Ears revisions | Big Ears generator built from scratch; ends unresolved (treble ear tip arc flips) | I |
| 46 | Panel A revision | POCKET 0.43 → 0.42 → 0.41; **C1/C2 bridging bug found**; v8; audits `Big_Ears_4.zip` | L |
| 47 | Troubleshooting cut file with jhg collaboration guidelines | `Big_Ears_5.zip` + `Panel_A_point_41mm.zip`; full 3-phase ClaudeCAM run; ARC_MAX_R → 200, samples 30 → 60 — **latest body session** | L |
| 48 | Importing NC files to Fusion 360 CAM | off-pipeline verification tools | M |

**Known conflict at #45/#46.** Content evidence says Big Ears was built before
Panel A revision (which audits `Big_Ears_4.zip`). Move-order evidence says the
reverse. One of the two is off by one slot. Everything downstream is unaffected.

---

# PHASE 6 — August (real timestamps)

| # | Session | Date |
|---|---|---|
| 49 | Migrating ClaudeCAM to persistent project | 2026-08-05 |
| 50 | ClaudeCAM project migration structure | 2026-08-05 |

---

# Unplaced

| Session | Why unplaced |
|---|---|
| Improving file documentation with external variables ("Dalek Destroyer") | Separate project — OpenSCAD acoustic cabinet, phyllotaxis spiral. No shared version markers. Recommend relocating out of this project. |
| Custom guitar prototype with EMG pickups ("The Offset") | Instrument design phase, predates CAM work. Contains EMG 89/89R placement dims (19.125" neck, 23.875" bridge @ 25.5" scale) that feed body geometry. |

---

# PROVENANCE TRACES

Trace-back index. Each chain reads oldest → newest.

## T1. Loopties / self-intersecting offsets

```
Converting SVG to 3D CNC cut files   root cause: Polygon.buffer() clips
  (Phase 0)                          peninsulas as concavities
        ↓
Reviewing the hand off               MITER_LIMIT 10 truncated horn by 25mm
  (Phase 1)                          → raised to 20
        ↓
Adjusting cut file for body fit      pocket corner tips filleted at
  (Phase 2)                          BIT_R + 0.2mm pre-offset
        ↓
Panel C Big Ears revisions           C1 discontinuity in SVG bezier →
  (Phase 5)                          opposing G3→G2 in finish pass
        ↓
Troubleshooting cut file...          arc fitter spans concave neck dip
  (Phase 5)                          → samples_per_curve 30 → 60
```

**If a loopty appears now**, check in this order: arc fitter sampling density,
bezier C1 continuity at the source, MITER_LIMIT, offset method.

## T2. POCKET_EXPAND / neck fit

```
Adjusting cut file for body fit     0.4  first expansion
Body cut file review (Mar 21)       0.4  "too tight"
Panel A tighten up                  0.5  "too loose" → 0.45
                                         sign convention documented
Cut file issues in updated version  0.43 cut on Panel 3
Panel A revision                    0.42 → 0.41
Troubleshooting cut file...         0.41 current
```

Larger value = looser fit (negated at call site). XY is `.3f`, Z is `.1f`.

## T3. Cut direction / winding

```
CNC coordinate centering            outward offset needs LEFT normal on a
  (Phase 0)                         CW-wound polygon in NC space
        ↓
Headstock Neo 3 work                CW (climb) reversed to CCW conventional
  (Phase 1)                         for MDF outside profiles
        ↓
CNC cutting feedback...             body outline point list reversed;
  (Phase 4)                         EMG vertices reversed for CCW interior
        ↓
Examining Panel C...                EMG winding reversal added — suspected
  (Phase 4)                         cause of one-side-always-climb chatter
        ↓
Cut file issues in updated version  UNRESOLVED: SVG path direction
  (Phase 5)                         C1→perimeter→C2 gives wrong winding
                                    for an outside profile
```

**This one has never fully closed.** Four separate sessions, each fixing a
different surface of the same confusion.

## T4. Arc fitting

```
Headstock Neo 3 work           greedy fitter + GRBL error 33 (I/J recompute)
SVG cut files w/ bezier...     ARC_MAX_R 200 → 100 working fix; 170mm radius
                               produced 393mm endpoint mismatch — ROOT CAUSE
                               NEVER FOUND
SVG design file handoff...     near-duplicate points (0.001mm) → 90° micro-jogs
                               → dedup in NC space at 0.05mm
CNC cutting feedback...        reverse_commands() swaps G2↔G3 with I/J recompute
                               (replaced naive arc→G1 flattening that gouged horns)
Troubleshooting cut file...    ARC_MAX_R restored to 200; samples_per_curve
                               30 → 60 instead
```

## T5. Handoff / continuity failures

```
File editing assistance          script claimed created, never existed
Building panel C handoff         handoff stale on a key point
Panel C workfiles analysis       handoff described a fix not actually
                                 implemented → audit-the-handoff protocol
Panel C Big Ears revisions       "three weeks of reintroduced bugs"
                                 → push working files to GitHub, fetch at
                                   session start, modify proven code
Cut file issues in updated ver.  → NC Delivery Gate (runbook 1_3)
                                 → 33-check automated verifier
```

The mitigations got progressively more structural. The last two are the ones
currently not in the public repo.

## T6. Pocket clearing

```
Pocket Clearing Algorithm         ROUGH_OFFSET boundary contract
Continuous maze pocket clearing   8 approaches fail; traversal ≠ geometry
Continuing the project            v2 walker; failed-approaches table
Algorithm for shape clearing      PIVOT: abandon walker, use offset rings
                                  gap end condition is shape-independent
Maze Walker handoff review        v4: Come CW / Go CCW
Maze walker idiot claude          v5: gap interpolated to exact s_adjusted
Maze Runner development progress  v6: Strategies A/B/C; Strategy C unproven
I/O error during file export      adaptive overlap search; inset-collapse
                                  as termination test; zero center lanes
Polishing MazeRunr                PTW-from-bbox is a shortcut; shapely
                                  medial_axis proposed; OPEN
```

---

# OPEN ISSUES, DATED

| Issue | First seen | Last touched | Status |
|---|---|---|---|
| Machine halt at horn tip, 0% progress bar | Phase 1 | Phase 1 | **Oldest open.** Isolation test file built, result never recorded |
| Oblong tuner holes / possible 6mm bit | Phase 0 | Phase 1 | Deferred for caliper measurement, still deferred |
| ARC_MAX_R degenerate fit root cause | Phase 4 | Phase 5 | Worked around twice, never diagnosed |
| Body outline winding on two-module path | Phase 4 | Phase 5 | Explicitly unresolved at last session |
| PTW from bbox vs medial axis | Phase 5 | Phase 5 | Open strategic question |
| Strategy C (crescent shapes) | Phase 5 | Phase 5 | Written, commented out, unproven |
| 5px close-point cleanup threshold | Phase 4 | Phase 4 | Deliberately not promoted to a rule |

---

# How to use this for trace-back

1. Identify the phase from the markers — filename convention, doc version cited,
   POCKET_EXPAND value, ring stitch version, presence of "ClaudeCAM."
2. Find the relevant T-chain above.
3. Everything downstream of the origin point in that chain is a candidate for
   impact. The Phase 4 → 5 boundary matters most: anything before Mar 28
   predates the runbook and the verifier, so it was neither gated nor checked.
