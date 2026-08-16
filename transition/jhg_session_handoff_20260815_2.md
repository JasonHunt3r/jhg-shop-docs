# JHG ClaudeCAM — Session Handoff, 2026-08-15 (second session)

Supersedes `jhg_session_handoff_20260815.md`, written the same morning. That
file is still accurate on the Panel C chord history and the traps, and is worth
reading for those. **It is now wrong about `ARC_MAX_R` and about open item A** —
see §1.

Same rule as its predecessor, from `jhg_troubleshooting_and_build_discipline_1_6.md`:
**this file is a communication tool, not a source of ground truth.** Items are
labeled Confirmed / Reported / Open. Verify Confirmed items before relying on
them if time has passed.

---

## 0. State of the repos — Confirmed

Both clean, both pushed, nothing uncommitted.

| Repo | HEAD | |
|---|---|---|
| `jhg-shop-docs` (public) | `ddb7c78` + this file's own commit | hygiene 2.0 is the live G-code authority; 1.9 retired |
| `jhg-shop-jobs` (private) | `3cc3115` | 5 commits this session |

**Do not re-audit the doc library.** Still current. Live set: troubleshooting
1.6, workflow 1.1, **gcode hygiene 2.0 (the G-code authority — renumbered from
1.9 this session)**, standards 2.0, plus the scale-widget / neck-pocket /
MazeRunr leaf docs.

Delivery gate and both fixture suites pass:

```
python3 verify_nc.py <file.nc>     # exit 0
python3 test_verify_nc.py          # gate fixtures       -- exit 0
python3 test_generators.py         # input guards (NEW)  -- exit 0
```

A fourth check now exists and is not run by any of the above — the virtual
cut, §2. Use it before a real cut, not instead of the gate.

---

## 1. What changed, and what the morning handoff now gets wrong

That file listed as open item A.2: *"Raise Panel A's `ARC_MAX_R` 100 → 200."*
**Closed — but not at 200, and not as a units fix.** It was the wrong framing.

### `ARC_MAX_R` is now derived — Confirmed

All four generators compute it instead of carrying a literal:

```python
ARC_MAX_R = (ARC_MAX_CHORD**2 / 4 + ARC_FIT_TOL**2) / (2 * ARC_FIT_TOL)  # 281.3mm
```

An arc is worth emitting only if its curvature is resolvable at the tolerance
the fitter works to. Above 281.3mm it departs from its own chord by less than
`ARC_FIT_TOL`, so G1 is as accurate and cheaper.

**The 100/200 split was never a decision.** Headstocks have shipped 200 since
2026-03-19. Body panels A *and* C were both born at 100 on 2026-03-23 — a clamp
to make one 170mm-radius degenerate fit unreachable, in the same session that
grafted the chord defect into Panel C. `Big Ears 6` restored C to 200 on
2026-04-06; Panel A never re-based. Full provenance is in hygiene 2.0 so this is
not re-derived a fourth time.

Effect: Panel A 6591 → **3676** NC lines, Panel C 10295 → **7058**. No accuracy
change — swept 50 → uncapped, deviation stays pinned at `ARC_FIT_TOL`.

### The sweep rule — the significant find, Confirmed

Hygiene has said *"sweep angle must stay under ~170 degrees"* since March. The
headstocks enforce it. **The body panels never did.** `fit_arc` bounded radius,
chord and tolerance — none of which limit how far around the circle a span runs.

Unbounded, it closes. Panel A at `samples_per_curve=60` fitted a span whose
endpoints landed **0.012mm apart on a 55.1mm-radius circle**. GRBL's rule for
start==end with I/J given makes that a **full circle** — a 110mm ring cut
through the part at `FEED_ROUGH`, at depth, once per depth pass.

| max fitted sweep | 15 | 30 | 60 | 120 | 240 |
|---|---|---|---|---|---|
| Panel A | 103° | **104° ships** | **360°** | 346° | 360° |
| Panel C | **285°** | **307°** | **87° ships** | 79° | 92° |

Now bounded in both body panels (`ARC_MAX_SWEEP_DEG = 170.0`) and gated by
`verify_nc.py`, which exempts hole-scale radii — HELICAL ORBIT uses deliberate
180°/360° arcs, and all 922 over-swept arcs in the archive are orbit geometry at
r=1.32mm. Shipped output was never affected, so both panels regenerate
byte-identical.

**This also settles `samples_per_curve`.** Panel A at 30 / Panel C at 60 was
**load-bearing, not drift** — each sat on the only value that avoided the defect
on its own geometry, and nothing recorded why. With the rule enforced, every
count from 15 to 240 is clean on both panels and sample density is a free
choice again. Do not "unify" it as a tidy-up; it no longer matters either way.

### Path fidelity — a new gate, Confirmed

**`ARC_FIT_TOL` is not a bound on path error.** It bounds error *at sample
points*; between them the arc is unconstrained. On Panel C at the old cap that
reached **0.345mm**, 3.5x the tolerance everyone assumed was binding.

It is invisible to the obvious measurement. Comparing target points to the
emitted path is the fitter's own acceptance test restated and returns
`ARC_FIT_TOL` no matter what the path does in the gaps. Only measuring points
sampled *along the emitted path* against the target curve sees it.

- `jobs/path_fidelity.py` measures it — **one shared implementation**, imported
  by both body panels, deliberately not copied per job.
- Generators wrap `arc_fit_points` so no call site can skip it, and stamp a
  `PATH FIDELITY` trailer into the NC. Current: Panel A **0.1167mm**, Panel C
  **0.1737mm**.
- `verify_nc.py` blocks above `PATH_DEV_LIMIT` (0.2mm = 2 × `ARC_FIT_TOL`).

### Two input guards — Confirmed

Both defend an assumption nothing upstream enforces, where being wrong yields a
smooth, closed, gate-passing NC that cuts the wrong shape.

1. **Panel A's bezier parser** handled absolute `M`/`C`/`L` through a `findall`
   that silently skipped everything else. A relative `c`, an `S` shorthand, an
   `H`/`V` or a `Z` would be dropped and the point list would jump the gap,
   leaving a straight cut across it at depth — the March chord signature. Which
   command set comes out is an Illustrator export setting. Now raises.
2. **Panel C's `offset_curve_outward_closed`** takes the left normal as outward,
   true only for a CW path. Reversed, every offset flips inward and the rough
   pass cuts 2mm *inside* the finish wall. Now asserts signed area < 0, pinned
   to the winding this panel has always shipped (−106930mm²).

---

## 2. The virtual cut — Confirmed, and worth using before the next real one

Jason installed **CAMotics 1.2.0** on 2026-08-14. Each body-panel job now
carries a `.camotics` project beside its NC with the real tool and stock
already set; open one in the GUI, or run it headless. Setup, the `camsim`
`libcairo` workaround, and the limits are in `CLAUDE.md` — do not re-derive
them, and in particular **do not reach for Homebrew** when `camsim` reports a
missing `/usr/local/lib/libcairo.2.dylib`. The app bundle ships its own copy;
nothing needs installing, and on Apple Silicon a Homebrew install would have
been the wrong architecture anyway.

**What it settled for the current pair — none of which this repo could check
on its own:**

- **A third-party interpreter reads our arcs identically.** CAMotics' engine
  traces the same path ours does to **0.007–0.010mm**, its own linearisation
  budget. Direction, I/J and sweep all agree. That is exactly the property the
  sweep defect (§1) violated, now independently confirmed rather than
  self-attested.
- **Nothing is cut outside the stock**, nothing below Z−15 but the stock
  bottom, and **the deliberate 1mm hold is present** in the solid.
- **The finished wall stands 3.170mm (A) / 3.171mm (C) from the finish
  toolpath**, against `BIT_R` 3.175. The part is the shape the toolpath
  intends.

**What it cannot settle.** 0.6mm voxels: topology and gross correctness, not
the 0.1mm-class wall accuracy `ARC_FIT_TOL` and `PATH_DEV_MAX` govern. Nothing
about feeds, chip load, MDF behaviour, bed flatness or workholding. **A clean
simulation is not a cut, in exactly the way a clean `verify_nc.py` is not a
cut** — same caveat, same reason, and it belongs on both.

*Expected and not a bug:* the part never separates in the simulation, because
`DEPTH_SCHEDULE` holds 1mm short until the bed is planed. It will look like the
profile failed to cut through. It didn't.

`gcodetool`, in the same directory, needs no workaround and is useful on its
own — `--linearize` expands arcs through that independent interpreter, which is
how the emission cross-check above was done.

---

## 3. Generator state — Confirmed

| | Panel A | Panel C | Panel E |
|---|---|---|---|
| ladder | 2.0 / 1.15 / 0.3 | 2.0 / 1.15 / 0.3 | 0.5 / 0.2 (Gen-0) |
| stepped penultimate | yes | yes | no |
| dedupe + loop excision | yes | yes | n/a (PyClipper) |
| finish-corridor guard | yes | **no** | no |
| `ARC_MAX_R` | **281.3 (derived)** | **281.3 (derived)** | – |
| `ARC_MAX_SWEEP_DEG` | **170** | **170** | – |
| path-fidelity stamp | **yes** | **yes** | no |
| input guards | parser | winding | no |
| `.camotics` sim project | **yes** | **yes** | no |
| `samples_per_curve` | 30 | 60 | – |
| `CORNER_RAMP_MM` | 5.0 | 3.0 *(correct — do not unify)* | – |
| current NC, gate-passing | yes | yes | **none, only in `history/`** |

`CORNER_RAMP_MM` differing is still **correct** — Panel C's EMG ear short sides
are 7.61mm, so a 5.0mm ramp from both corners leaves no nominal-feed run.

**Panel A's pocket architecture is intact and was verified, not assumed.**
`POCKET_EXPAND` is applied once, only to the green pocket U, upstream of arc
fitting; pocket and body are then stitched into single runs, so one section
carries the adjustment on part of its length only. Probed by zeroing
`POCKET_EXPAND`: EMG sections move 0.0000mm, the combined sections show median
0.0000 with max 0.42 — identical before and after this session's changes.

---

## 4. Open items

**A. Direction consistency — Open, the closest sibling of what was just fixed.**
The headstocks check that a span does not flip CW/CCW mid-arc ("an inflection
point is not a single arc" — hygiene, ARC FITTING CONSTRAINTS). **The body
panels do not.** Same lineage gap, same shape, same doc, still unimplemented.
This is the highest-value remaining item precisely because the sweep rule turned
out to matter.

**B. Headstock reissue — Open, small but has a decision in it.** Both headstock
generators carry the derived `ARC_MAX_R` but were **not** regenerated;
`headstock-neo4/jhg_headstock_neo4_verified_3.nc` is a known-good and reissuing
it is Jason's call. Their next output will differ from it.

**C. Promote the missing-stamp advisory to blocking — Open, do it after B.**
An absent `PATH_DEV_MAX` is currently advisory, because the archived NCs and
both frozen fixtures predate it. Once every generator has reissued, make absence
blocking. That is what closes the loop the same way the chord gap was closed.

**D. Port `truncate_at_finish_crossing` to Panel C — Open, unchanged.** Was A.1
in the morning handoff. Belt-and-braces; C has no crossings for it to catch.

**E. Panel E — Open, still needs Jason's decision first.** Its header says
"Outline-only variant of Panel C. No EMG pocket cuts." Either it is still wanted
(needs the current ladder, the derived constants, the guards, and a regenerate)
or Panel C superseded it (retire to `archive/`). Don't align it until answered.

**F. No tool select in any NC — Open, small, note only.** The files issue no
`T`/`M6`, so a simulator or controller has to guess which tool is loaded —
CAMotics warns *"cutting move but no current tool, selecting tool 1"*. Harmless
on GRBL, which has no changer and cuts with whatever is in the spindle, but the
header block could state the intended tool so a reader and a simulator agree.
Found by the virtual cut, §2.

**G. Dead code in Panel A — Open, note only.** `offset_curve_outward` and
`offset_open_curve_outward` are defined and never called; Panel A offsets
through `offset_open_curve_inward` and `offset_polygon_outward`. Left in place
deliberately (outside what was asked) but they are an attractive nuisance.

**H. Questions only a cut can answer — Open by design.** Unchanged from the
morning handoff: is `ROUGH_LEAVE = 2.0` sufficient on MDF (labeled Conjecture),
do the walls come off smooth enough, and the standing prediction of **no visible
scar at C1**. Nothing has been cut since 2026-03-25. The virtual cut (§2)
narrows this list but does not shorten it: it rules out the two failures most
worth fearing on a first cut after five months — a rogue arc, and walls in the
wrong place — and speaks to none of the material questions.

**I. Deferred** — repo visibility, skills triage.

---

## 5. Traps — carried forward, plus one earned this session

The morning handoff's list all still applies. Read it. Added:

- **Never run a generator against the repo path while debugging.** Generators
  write their NC next to the input SVG. Debug runs at forced parameters
  overwrote the repo NCs this session, and a later scan of those files produced
  a confident, wrong report that shipped Panel A contained 360° arcs. It did
  not. **Copy generator + SVG into a scratch directory first.** Restore with
  `git checkout --` and re-verify before believing any scan.
- **A metric that agrees with the thing it is checking is not a check.**
  Measuring target points against the emitted path returns `ARC_FIT_TOL`
  forever, because that is the fitter's own acceptance test. Two separate
  conclusions in this project were built on it. Ask what a measurement *cannot*
  see before trusting a clean result.
- **A rule implemented in one lineage is not implemented.** The sweep rule was
  written, enforced in the headstocks, and absent from the body panels for five
  months. Grepping the docs finds the rule; only grepping the generators finds
  out whether it runs.
