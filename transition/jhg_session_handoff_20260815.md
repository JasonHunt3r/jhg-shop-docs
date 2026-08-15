# JHG ClaudeCAM — Session Handoff, 2026-08-15

Written for a fresh Claude Code instance. Per this project's own rules
(`jhg_troubleshooting_and_build_discipline_1_6.md`, HANDOFF VERIFICATION):
**this file is a communication tool, not a source of ground truth.** Items are
labeled Confirmed / Reported / Open. Verify Confirmed items still hold before
relying on them if time has passed.

---

## 0. State of the repos — Confirmed

Both clean, both fully pushed, nothing uncommitted anywhere.

| Repo | HEAD | Notes |
|---|---|---|
| `jhg-shop-docs` (public) | `2831b0c` | doc library rewritten this week |
| `jhg-shop-jobs` (private) | `4d046b9` | generators + `verify_nc.py` |

**Do not start by re-auditing the doc library.** It was rewritten 2026-08-14
to one-home-per-rule and is current. Live set: troubleshooting 1.6, workflow
1.1, **gcode hygiene 1.9 (the G-code authority)**, standards 2.0, plus the
scale-widget / neck-pocket / MazeRunr leaf docs. The runbook is retired.

---

## 1. Read this before investigating any Panel C "incident" — Confirmed, CLOSED

This has now been re-derived **twice** by sessions that didn't check a rule's
status line. Do not make it three.

**What happened:** on **2026-03-25** Jason hit the emergency stop on a Panel C
cut. The tool rapided to the heel, plunged to Z-16, and set off on a 399mm
straight line across the body.

**Mechanism:** `ARC-FIT PATH ROTATION` — the arc fitter started at index 0
while the plunge point came from a separate marker, so the first cutting move
after each plunge was a chord to wherever index 0 landed. Documented in
`jhg_gcode_hygiene_1_9.md`, **status CONFIRMED FIXED 2026-08-13**.

**Provenance (established 2026-08-15):** the defect was *introduced*, not
inherited. Archived Panel C NCs scan clean on 03-19; the chord first appears
**2026-03-23** in the session that synced Panel C's ladder to Panel A's
standards — Panel A's structure grafted on. Persisted into
`jhg_body_pnlC_rot17_2.nc` (03-25 17:11), the file that was stopped.

**The fix shipped the same day.** `Proven Cuts/Panel C proven/jhg_body_pnlC_rot17_3.nc`
(03-25 17:31) has zero long moves at depth and passes `verify_nc.py`. It was
simply never cut — shop time ran out, and **nothing has been cut since
2026-03-25**. The April work was all generation, no cutting.

**Three phrasings that misled earlier sessions, all corrected:**
- It was **not a jam.** Jason e-stopped it; nothing stalled, no bit broke.
  "It jammed my machine" was shorthand for "it could have."
- It was **not April.** 2026-03-25.
- **"Cut into the pocket"** meant the region where *Panel A* has a pocket —
  Panel C's solid **heel**. No Panel C file has ever had a pocket section.

**The real lesson:** the rule was written the same day, the fix was made the
same day, and the good file was filed correctly. What broke was the loop from
*rule written* → *rule verified against the file that ships*. The older
working lineage kept the bug until 2026-08-13 — one defect, fixed twice, in
two branches unaware of each other. That gap is what `verify_nc.py` closes.

---

## 2. The delivery gate — Confirmed

`jobs/verify_nc.py` — standalone, **exits non-zero**, so delivery is gated by
a return code rather than by a rule someone remembers.

```
python3 verify_nc.py <file.nc> [...]      # -v adds advisories
python3 test_verify_nc.py                 # fixtures must hold
```

Blocking: self-intersection (angle-discriminated), arc radius consistency,
per-pass engagement depth, code-portion format hygiene. Advisory:
first-move-after-plunge, file size.

Two fixtures pin the two defect classes the machine has actually been exposed
to. Both must keep failing:
- `jhg_headstock_neo3_tpl_1.nc` → **271** radius mismatches, matching the
  March incident write-up exactly.
- `jhg_body_pnlC_rot17_norough_DONOTRUN.nc` → engagement findings. This file
  is kept byte-identical as evidence. **DO NOT RUN IT.**

Calibrated against all 94 archived NCs: 66 pass / 28 fail, every failure a
real documented defect class.

---

## 3. Generator state — Confirmed

| | Panel A | Panel C | Panel E |
|---|---|---|---|
| ladder | 2.0 / 1.15 / 0.3 | 2.0 / 1.15 / 0.3 | 0.5 / 0.2 (Gen-0) |
| stepped penultimate | yes | yes | no |
| dedupe + loop excision | yes | yes | n/a (PyClipper) |
| finish-corridor guard | yes | **no** | no |
| `ARC_MAX_R` | **100** | 200 | – |
| `CORNER_RAMP_MM` | 5.0 | 3.0 *(correct — see below)* | – |
| current NC, gate-passing | yes | yes | **none, only in `history/`** |

**Panels A and C are a matched, gate-passing pair — this is the starting
point.** Neither can reproduce the chord: C rotates to the plunge index
explicitly, A sets `plunge_idx = 0` and loops from it by construction.

`CORNER_RAMP_MM` differing is **correct, do not unify** — Panel C's EMG ear
short sides are 7.61mm, so a 5.0mm ramp from both corners would leave no
nominal-feed run between them.

---

## 4. Open items

**A. Panel A/C alignment — Open, small, do these together**
1. Port `truncate_at_finish_crossing()` from Panel A to Panel C. Belt-and-braces
   only — C currently has no crossings for it to catch.
2. Raise Panel A's `ARC_MAX_R` 100 → 200. *Reported:* 200 is the documented
   value and what headstocks and Panel C ship; Panel A's 100 has no recorded
   reason. **Measured:** 72.5% of Panel C's arcs fall in the 100–200mm band —
   real body-curve geometry that Panel A is currently linearizing. Requires
   regenerate + re-verify + diff, not a blind edit.

**B. Panel E — Open, needs Jason's decision first.** Its own header says
"Outline-only variant of Panel C. No EMG pocket cuts." Either it's still a
wanted operation (needs the current ladder and a regenerate) or Panel C
superseded it (retire to `archive/`). Don't "align" it until that's answered.

**C. Questions that only a cut can answer — Open by design**
- Is `ROUGH_LEAVE = 2.0` sufficient on MDF? Labeled **Conjecture**; the cut
  that would have answered it was the one that got stopped. Cheap either way —
  a wrong leave costs sanding time, not part dimensions.
- Do the walls come off smooth enough that sanding stays light?
- Prediction on record: **no visible scar at C1.** Both defects found this week
  are toolpath-integrity anomalies with zero dimensional harm (0.14mm sits
  inside the finish bit's own 3.175mm swath). Absence of a mark confirms the
  model; it does not decide anything.

**D. Deferred, not blocking** — repo visibility (public is load-bearing for
App Claude's unauthenticated pull; Jason's call), skills triage (wants its own
session; constraint: docs must stay flat and greppable).

---

## 5. Traps that have already cost time

- **Check a rule's status line before investigating.** A stale "root cause
  unresolved" note in memory sent this session hunting a solved problem.
- **Grep the doc library for the mechanism before diagnosing.** Standing rule;
  broken twice by the same bug.
- **Jason's informal incident language is approximate.** Ask what the words map
  to before building a search on them.
- **A finding's tier must not exceed its evidence's reach.** Path analysis
  confirms geometry, not consequence.
- **Endorsement is not evidence.** Agreement between Claude instances doesn't
  promote a Conjecture.
- **Stage commits explicitly.** `git add -A` twice swept unreviewed files in.
- **Verify writes on disk before claiming them.**

---

## 6. Working with App Claude

App (claude.ai) holds the 52-conversation record; this instance holds the
files. **App tells you *why* a value exists; CC tells you *what ships*. App's
provenance is a lead, not a finding, until confirmed against disk** — several
of its claims were overturned by file checks this week, and several of this
instance's claims were overturned by its records. The division works because
each side's blind spot is the other's evidence.

`Evolution/MANIFEST.md` and `Evolution/PARAM_CENSUS.md` are the join key —
upload them to the App project so it can search by filename and relay paths
back. Write files to `Evolution/for-app-upload/` in plain ASCII rather than
pasting; Unicode-heavy paste corrupts.

---

## 7. Local archive — not in any repo

`Evolution/` (gitignored, ~500 files, 2026-02-28 →) organized by component.
**The 34 zips and 6 `ƒ` folders are session-end snapshots** — the files inside
one belong together as a coherent state. Move them whole; never open-and-scatter.
`~/Desktop/claude docs/` holds per-session checkpoints for all Jason's projects
(much is unrelated to JHG) and `Proven Cuts/`, his own known-good picks —
which, verified this week, does hold the corrected Panel C file.
