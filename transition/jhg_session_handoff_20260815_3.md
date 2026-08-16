# JHG ClaudeCAM — Session Handoff, 2026-08-15 (third session)

Supersedes `jhg_session_handoff_20260815_2.md`. That file remains accurate on the
derived `ARC_MAX_R`, the sweep rule, path fidelity, the input guards and the
CAMotics virtual cut — read §1 and §2 there for those. **Its open-items list is
superseded by §3 below**, where four of its nine entries are now closed.

Same rule as its predecessors, from `jhg_troubleshooting_and_build_discipline_1_6.md`:
**this file is a communication tool, not a source of ground truth.** Items are
labeled Confirmed / Reported / Open. Verify Confirmed items before relying on them
if time has passed.

---

## 0. State of the repos — Confirmed

Both clean, both pushed, nothing uncommitted.

| Repo | HEAD | |
|---|---|---|
| `jhg-shop-docs` (public) | `df96a9c` + this file's own commit | 7 commits this session |
| `jhg-shop-jobs` (private) | `e7bfcb8` | 5 commits this session |

**Do not re-audit the doc library.** Live set unchanged: troubleshooting 1.6,
workflow 1.1, **gcode hygiene 2.0** (the G-code authority), standards 2.0, plus the
scale-widget / neck-pocket / MazeRunr leaf docs.

All four checks pass:

```
python3 verify_nc.py panel-a/jhg_body_pnlA_rot17_overlays_8_1.nc   # exit 0
python3 verify_nc.py panel-c/jhg_big_ears_pnlC_overlays_3.nc       # exit 0
python3 test_verify_nc.py                                          # exit 0
python3 test_generators.py                                         # exit 0  (12 guards)
```

The virtual cut (CAMotics) is a fifth check, not run by any of the above. Setup and
limits are in `CLAUDE.md`. Use it before a real cut, not instead of the gate.

---

## 1. What changed this session

**Direction consistency enforced in both body panels** (item A). The third and last
of the arc-fitting rules the headstocks had and the body panels didn't. Panel C
byte-identical; **Panel A changed at 15 spans** — 10 in body outline rough, 5 in
penult step 1, **none in finish or spring** — where the fitter was swallowing an
inflection into a single 69.4mm arc. `PATH_DEV_FROM` unchanged at 0.1167mm; a
CAMotics run before and after voxelises to an identical 2,464,810-triangle surface.
**Not gateable by `verify_nc.py`** — an emitted G2/G3 is a single arc by definition,
so the defect is only visible against the point list the NC does not carry. Pinned
by `test_generators.py` instead.

**The finish-corridor invariant was removed, not ported** (item D). See §2 — it is
the one thing in this session worth reading in full before touching Panel A or C.

**Tool named in both NC headers** (item F). Bit 1 identified from its case:
`SPWS2LX6.22`, 6.35 dia x 22 LOC x 50 OAL x 6.35 shank, 2-flute. The **22mm flute
length clears `DEPTH_TOTAL` 16.0mm** with 6mm to spare — nobody had checked.
Comment-only: `T1 M6` was measured to silence CAMotics and yield an identical
surface, but it has never been sent to the DLC32 and Jason chose not to emit it.
**"Upcut" is Jason's attestation — the case does not state it.** Do not "correct"
it from the photo.

**Two dead offset functions removed from Panel A** (item G). Byte-identical
regeneration. `offset_open_curve_outward`'s docstring claimed it drove the body
outline offsets; the body has always used `offset_open_curve_inward`, so a reader
trusting it would have had the offset direction of the whole body backwards.

**A self-check false alarm fixed.** Both body generators' `End-of-program sequence`
check counted the `PATH FIDELITY` trailer stamped after `M2`, and printed
`*** DO NOT RUN THIS FILE ***` on every correct file. Comment lines are now excluded.

**`reference-images/`** — new gitignored, local-only depository for photos of
physical shop things. Top level = unread; `transcribed/` = content is already in a
doc and the image is only evidence. Convention is in `CLAUDE.md` because the
folder's own README is untracked. **Nothing in the docs may depend on an image
being present.**

---

## 2. The finish-corridor invariant — removed. Read before re-adding anything like it.

Panel A briefly carried `truncate_at_finish_crossing()`, enforcing *no rough or
penultimate path may cross the finish geometry*. It was **removed 2026-08-15**, and
the rule is not part of this methodology.

- **No provenance.** In no version of hygiene, standards or workflow, and App Claude
  found no decision to adopt it anywhere in the 52 archived claude.ai conversations.
  Introduced by a Claude Code session on 2026-08-14 (jobs `9770916`).
- **Its motivating measurement was retracted in the commit that added it** —
  claimed 0.311mm, measured **0.0075mm**, about 40x smaller.
- **Total effect on Panel A:** one point, the last point of `pocket_penult`, moved
  0.0075mm, **on the offcut side**. Two NC lines. An order of magnitude below
  `ARC_FIT_TOL` (0.1mm) and this file's own path deviation (0.1167mm).
- **It was not clipping the pocket approach.** Panel A's straight-line entry is a
  ~58mm `G1` run and is byte-identical with the rule on or off. The interleaved
  pocket+outline design dates to **2026-03-23** (`JHG Body Panel A revised.zip`) and
  that entry has held to within 0.09mm ever since.
- **A raising version was considered and rejected:** at 0.0075mm it would fire on
  every run of correct geometry, and a guard that always trips is not a guard.

The hazard it reached for is real — a bad SVG re-export or leave-ladder edit driving
a stock pass through the finish wall is dimensional and unrecoverable — and stays
covered by `verify_nc.py`'s `check_engagement_depth`, on the emitted file, which is
the right layer.

**The function is preserved**, verbatim and with all measurements, call sites and
restore instructions, at `jobs/archive/truncate_at_finish_crossing_retired.py`.
Restoring it is a paste, not a rewrite. Full reasoning: hygiene 2.0 → ARC FITTING
CONSTRAINTS; general lesson: troubleshooting 1.6 → AN UNDOCUMENTED INVARIANT IS A
HYPOTHESIS, NOT A RULE.

---

## 3. Open items — the list

**Everything that could be done at the desk is done.** Three of the five live items
are downstream of the same event: cutting a Panel C.

| | Item | Blocked on | Size |
|---|---|---|---|
| **B** | Headstock reissue | a proven Panel C cut | small, has a decision in it |
| **C** | Promote missing `PATH_DEV_MAX` to blocking | B | small |
| **E** | Panel E | a proven Panel C cut | small once C is proven |
| **H** | The material questions | a real cut | — |
| **I** | Deferred: repo visibility, skills triage | Jason | — |

**B. Headstock reissue — Open, Jason's decision, parked.** Both headstock generators
carry the derived `ARC_MAX_R` but were **not** regenerated, so their next output will
differ from `headstock-neo4/jhg_headstock_neo4_verified_3.nc`. **That file is a
proven, cut file Jason does not want to lose** — a reissue would sit alongside it,
never replace it. He is willing to run one later; it stays parked until a Panel C is
cut.

**C. Promote the missing-stamp advisory to blocking — Open, chained behind B.** An
absent `PATH_DEV_MAX` is advisory today because the archived NCs and both frozen
fixtures predate it. Once every generator has reissued, make absence blocking. That
closes the loop the same way the chord gap was closed.

**E. Panel E — Open, resolved in principle, waiting on the same cut.** Not "update
the stale generator": it is **the proven Panel C with the EMG paths deleted**. Its
header already reads "Outline-only variant of Panel C, no EMG pocket cuts." So it is
downstream of a proven Panel C, not a separate line of work.

**H. Questions only a cut can answer — Open by design.** Is `ROUGH_LEAVE = 2.0`
sufficient on MDF (labeled Conjecture); do the walls come off smooth enough that
sanding stays light; and the standing prediction of **no visible scar at C1**.
Nothing has been cut since **2026-03-25**. The virtual cut narrows this list without
shortening it: it rules out a rogue arc and walls in the wrong place, and speaks to
none of the material questions.

**I. Deferred.** Repo visibility — `jhg-shop-docs` is public as a claude.ai-era
holdover, but public is now load-bearing for App Claude, whose tarball pull is
unauthenticated. And skills triage, which wants its own session; constraint if it
happens: **docs must stay flat and greppable** or CHECK EXISTING RULES BEFORE
DEBUGGING breaks.

**Closed this session:** A (direction consistency), D (finish-corridor — by removal,
**explicitly not by porting**), F (tool select — comment-only), G (dead code).

---

## 4. Traps — carried forward, plus three earned this session

Everything in `_2` §5 still applies: never run a generator against the repo path
while debugging (copy to scratch); a metric that agrees with the thing it is
checking is not a check; a rule implemented in one lineage is not implemented.
Added:

- **Compare points, not counts — and let the diff arbitrate.** A probe reported
  `truncate_at_finish_crossing` "never fires, 0 points removed." It fires: a
  truncation in the **final segment** returns the same point count with the last
  point replaced. It was caught only when regenerate-and-diff contradicted the
  prediction of byte-identical output. When a prediction and a diff disagree, the
  diff is right.
- **Point-in-polygon is invalid against `body_finish`.** It is an **open** path with
  a 57mm gap between first and last point, so any inside/outside test silently
  closes it with a 57mm chord and returns a confident wrong answer. Use a local
  signed-side test against the nearest segment, with a known-side reference —
  `body_rough` is offset 2.0mm outward and is therefore known offcut.
- **State the measurement, not the narrative it supports.** The removal in §2 was
  correct on its own merits. The first write-up additionally claimed the rule was
  "clipping the deliberate junction," which the measurement did not support. A
  correct decision with an overstated justification still puts a false claim in the
  library, where the next session inherits it as fact. Corrected in docs `e1145fe`.
- **A fixture that cannot fail is not a fixture.** The first `test_generators.py`
  direction case looked like it worked — the backtracking span was rejected — but
  the **sweep** bound was rejecting it before the new code ran. Assert the
  pre-patch behaviour (`git show HEAD:...`) when adding a guard fixture, or it
  silently tests nothing.
