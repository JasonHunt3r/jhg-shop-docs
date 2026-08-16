# JHG ClaudeCAM — Session Handoff, 2026-08-16

Supersedes `jhg_session_handoff_20260815_3.md`. That file remains accurate on the
finish-corridor removal (its §2 — read it before re-adding anything like it) and on
the traps in its §4, all of which still apply. **Its open-items list is superseded
by §4 below.**

Same rule as its predecessors, from `jhg_troubleshooting_and_build_discipline_1_6.md`:
**this file is a communication tool, not a source of ground truth.** Items are
labeled Confirmed / Reported / Open. Verify Confirmed items before relying on them
if time has passed.

**Nothing was cut this session.** Nothing has been cut since 2026-03-25.

---

## 0. State of the repos — Confirmed

- **docs repo** — clean. This file is the only addition.
- **jobs repo** — clean, HEAD `87712ab`, **one commit ahead of `origin/main`, unpushed.**
- `Evolution/` and `reference-images/` — gitignored, local only, unchanged in structure.
- `Evolution/for-app-upload/` gained three CC status docs this session
  (`cc_emg_timeline_`, `cc_emg_round2_`, `cc_emg_round3_20260816.md`) plus App's two
  appended replies in `Evolution/` root. ASCII-verified; App's copies are the ones
  carrying the log-side answers.

---

## 1. What changed in the code

**One commit, `87712ab` — dead-code sweep on both body panels.** Removed:
`emit_side_with_ramp()` (33 lines, never called, docstring contradicted `loop_from`),
`bridge_fwd()` (a `pass` stub), `points_to_svg_path()`, Panel C's
`offset_open_curve_inward()` (dead there, still live and load-bearing in Panel A),
`prev_idx`, an unused `xform` parameter, identical if/else branches at the `M/m`
handler, a duplicate endpoint assignment in `reverse_commands`, and both hardcoded
`Generated: 2026-04-0x` header lines.

**Gate held.** Both generators regenerated and byte-diffed against their prior NC.
Only delta in either file:

```
PANEL A NC:  13d12  < ; Generated: 2026-04-04
PANEL C NC:  13d12  < ; Generated: 2026-04-05
```

Both preview SVGs byte-identical. `verify_nc.py` PASS on both, `test_generators.py`
12/12, `test_verify_nc.py` 12/12.

**No timestamp is emitted into the NC now, deliberately.** A wall-clock stamp makes
every run differ from the last and would retire the byte-diff regeneration gate
permanently. The frozen April date did not earn that gate, it merely failed to break
it. If provenance is wanted in the header later, it belongs as a generator version or
git SHA.

### The depth block — read this before touching it

**Both panels now state, in identical words, that the last depth pass is temporarily
disabled on purpose.** Holding 1mm short leaves a thicker floor, which **keeps the
workpiece secure through the edge finishing passes**. The bed is not flat, so a
full-depth pass thins the onion skin unevenly and it tears — **that has already cost
a cut.**

`FULL_DEPTH` and the commented-out true-depth `DEPTH_SCHEDULE`/`FULL_DEPTH` pair
**stay**. The commented lines are the restore path. An earlier pass this session
deleted them as unread; that was wrong and was reverted, and the commit was **amended
rather than followed by a revert commit** so the history reads as the decision rather
than as a stumble. Before this session Panel A explained only the bed — never the
purpose or the failure — and **Panel C explained nothing at all.**

---

## 2. `reverse_commands` — reclassified, and now a compliance item

Traced across all 33 archived body-panel generators.

| Date | Signature | idx0 | File |
|---|---|---|---|
| 03-23 02:34 | `(commands, start_point)` | emit | panel-a born with it |
| 03-23 04:43 | `(commands, start_point)` | emit | **panel-c, ported from A, intact** |
| 04-04 15:59 | `(commands)` | emit | **panel-a drops `start_point`** |
| 04-04 17:22 | `(commands)` | DROP | **panel-a adds the `pass`** |
| 04-05 04:32 | `(commands)` | DROP | panel-c inherits the degraded version |
| CURRENT | `(commands, start_point)` | emit | jobs/panel-a — August **restored** it |
| CURRENT | `(commands)` | DROP | jobs/panel-c — never restored |

So the `pass` is **not** original design and **not** contamination residue. It is a
Panel A regression of 2026-04-04 that reached Panel C by copy the next day.

`jhg_gcode_hygiene_2_0.md`, GO-AND-COME ARC REVERSAL, states the rule:
*"The `reverse_commands()` function takes the forward command list and the forward
path's start point."* **Panel A complies. Panel C does not.** Panel A was brought
into compliance on 2026-08-13 — flagged then as "code not matching an existing
correct doc rule" — and nobody checked Panel C. That is BUG CLASS RECURRENCE in its
usual form.

Measured effect: **0.483mm** of come-leg not re-cut on the FINISH body lap,
surface-quality class, closed by the next section's G1 in every case but the final
spring lap. Fixing it **restores Panel C's own 2026-03-23 behaviour**.

Open. Jason's call. Not applied.

---

## 3. Questions closed by measurement — do not re-open without new evidence

- **`MITER_LIMIT` is a dated fossil.** PyClipper was live in
  `panel-a/jhg_body_pnlA_v5_rot17.py` on 03-20 (`pco.MiterLimit`, `JT_MITER`,
  `ET_CLOSEDPOLYGON`) and was **deliberately retired 03-23** in the rewrite that
  created `emit_emg_aperture`. Unread in every body-panel generator since. **Kept and
  annotated in both panels — do not delete.**
- **The A/C Clipper migration is documented open work, not a misreported fix.**
  `jhg_shop_file_standards_2_0.md` §319 already says so, and names the orphaned
  constant. `jhg_session_handoff_20260815_2.md` §179 records two remedies for two
  code paths — headstocks on Clipper, A/C on dedupe + excision. Nothing was reported
  as done that wasn't.
- **EMG aperture loops: measured clean.** Zero self-intersections in either aperture
  on either panel at every ladder value (2.0 / 1.15 / 0.3), run through the
  generators' own offset functions on real SVG geometry. `excise_self_intersections`
  never reaching `offset_polygon_outward` is a real structural gap **with no current
  instance.**
- **One residual crossing exists, in Panel A body rough at 2.0** — X-88.015 Y169.695,
  post-excision, reaching the NC at all 10 depth levels. It is a **0.0932mm backtrack
  at a 2.23° crossing angle**. `verify_nc.py` sets `LOOP_ANGLE_MIN = 10.0` — *"below
  this a crossing is a retrace artifact."* The gate is classifying it correctly. **Not
  a defect. Recorded so it is not rediscovered as one.**
- **`verify_nc.py`'s `break` is fine.** `_cross_angle` returns None when segments *do
  not cross*, so `continue` carries the scan across the full `LOOP_WINDOW`; the
  `break` fires only after a real crossing and means "nearest crossing per starting
  segment." Verified three ways: the `_1_4` fixture is caught
  (`X-105.50 Y160.13, 56 deg`), corpus results identical with the break removed, and
  synthetic loops at spans 3–80 identical. **No change recommended.**
- **The 2026-03-25 stopped cut: one of four defects was live.** Only the arc-fit path
  rotation. EMG diagonals absent, offset direction correct (+2.056mm vs
  `ROUGH_LEAVE` 2.0), the SAFE_Z item is Panel A structure Panel C does not have.
  **Jason's shop account confirmed by measurement.**
- **There are six chord moves, not two** — ROUGH ×1 (402.3mm @ Z-1.5), PENULTIMATE
  ×2 (399.4 @ Z-16), FINAL ×2 (398.4 @ Z-16), SPRING ×1. The shallow one executes
  first.
- **The 2026-03-19 Panel C NC is not a stopped-cut candidate** (91K vs the stopped
  file's 309K). Closed.
- **The 04-05 Big Ears episode is a byte-exact revert**, SHA256 `9857b5ae...`
  identical between the 04:32 loose file and `Big Ears 3.zip`'s member. It is **not**
  the stopped-cut episode; that was 03-25, eleven days earlier, different lineage.

---

## 4. Open items — the list

Everything at the desk that could be closed this session was closed. What remains is
either Jason's decision or downstream of cutting a Panel C.

| | Item | Blocked on | Size |
|---|---|---|---|
| **J** | Panel C `reverse_commands` compliance | Jason's call | small, behavioural |
| **B** | Headstock reissue | a proven Panel C cut | small, has a decision in it |
| **C** | Promote missing `PATH_DEV_MAX` to blocking | B | small |
| **E** | Panel E | a proven Panel C cut | small once C is proven |
| **H** | The material questions | a real cut | — |
| **K** | Six-chord physical check | Jason, shop-side | free |
| **L** | Clipper offsets for A/C | scheduling | large, measured yield zero |
| **I** | Deferred: repo visibility, skills triage | Jason | — |

**J. Panel C `reverse_commands` — Open, Jason's call.** See §2. Compliance with a
live doc rule, restores C's own original behaviour, 0.483mm surface-quality effect.
Behavioural change: it will move the NC, so it needs its own regenerate-and-verify,
not a byte-identical gate.

**K. Six-chord physical check — Open, shop-side, costs nothing.** If the stopped
panel still exists, look for **two roughly parallel scars about 1.5mm apart in Y**,
one shallow and one deep — `OUTLINE: ROUGH` at Z-1.5 running to X-4.343 Y-209.232,
and `OUTLINE: PENULTIMATE` at Z-16.0 to X-4.329 Y-207.732. Two scars confirms the
six-chord count physically and explains the sequence. One scar means the count or the
section ordering needs re-examining. This is the only physical evidence available for
an event otherwise reconstructed entirely from files.

**L. Clipper offsets for A/C — Open, scheduled after the body template.** Measured
yield: **zero** self-intersections it would remove from apertures at any ladder
value, and one 2.23° retrace in Panel A's body outline that is not a loop. Changing
the offset method changes every toolpath on both panels — a dimensional-class change
for no measured defect. Do not do this before a cut.

**B, C, E, H, I** — unchanged from `20260815_3` §3. All still blocked on the same
event: cutting a Panel C.

**Dropped:** Q4, the rationale for the 03-23 PyClipper retirement. Archaeology, and
it only looked live while the Clipper question looked live.

---

## 5. Traps — carried forward, plus four earned this session

Everything in `20260815_3` §4 and `_2` §5 still applies: never run a generator against
the repo path while debugging; a metric that agrees with the thing it is checking is
not a check; a rule implemented in one lineage is not implemented; compare points, not
counts, and let the diff arbitrate. Added:

- **A comment marking something deliberate, temporary, or disabled takes it off the
  deletion list.** "Never read" is an argument about the code; the comment is an
  argument about the shop, and the shop wins. This cost real time this session: an
  unread constant was removed from inside a documented safety workaround, and then the
  commented-out *restore path* was removed after the objection. A category-level
  go-ahead ("run the sweep") is not item-level approval for anything inside such a
  block.

- **A sweep that cannot fail is not evidence.** A corpus scan bounded with `timeout`
  produced a clean all-zeros table because **`timeout` is not installed on this Mac**
  — every invocation failed silently and grep counted nothing. It was caught only
  because a known-FAIL fixture sat in the same run. **Every corpus sweep must include
  at least one fixture whose expected result is non-zero.**

- **Open the live doc before characterising what it says.** Both Claudes got the
  Clipper question wrong from opposite directions this session — one declared a live
  doc stale, the other declared a fix missing — and `jhg_shop_file_standards_2_0.md`
  had said "open work" and "the plan that never landed" all along. A summary of a doc,
  or an inference about it, is not the doc.

- **Before calling a gate broken, run it against a fixture it is known to catch.** A
  one-line misreading of `continue` vs `break` in `verify_nc.py` was reported as a
  structural defect and promoted to top priority by the other party before the
  `_1_4` fixture — which the gate catches correctly — was ever run against it.

---

## 6. Where to start

1. Read `jhg_troubleshooting_and_build_discipline_1_6.md` (collaboration protocol),
   then this file's §4 and §5.
2. If the question is about EMG history, `MITER_LIMIT`, the stopped cut, or the
   Big Ears lineage: `Evolution/for-app-upload/cc_emg_timeline_20260816.md` and its
   round-2 and round-3 companions, plus App's appended replies in `Evolution/` root.
   Those carry the measurements and the retractions on both sides.
3. **Do not re-open anything in §3 without new evidence.** Several of those were
   re-derived more than once already.
