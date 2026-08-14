# JHG ClaudeCAM — Session Handoff, 2026-08-13

Written for a fresh Claude Code instance picking this up. Per this project's own
rules (`jhg_troubleshooting_and_build_discipline_1_5.md`, HANDOFF VERIFICATION):
**this file is a communication tool, not a source of ground truth.** Items below
are labeled Confirmed / Reported / Open. Verify Confirmed items still hold before
relying on them if much time has passed; treat Reported items as leads to check
against real files, not facts; Open items are genuinely unresolved.

---

## 1. Repo state (Confirmed)

- **Public** `github.com/JasonHunt3r/jhg-shop-docs` — the `.md` methodology docs,
  unchanged in nature, kept public only because that was originally needed for
  claude.ai URL-fetching (no longer relevant in Claude Code, but harmless to
  leave public).
- **Private** `github.com/JasonHunt3r/jhg-shop-jobs` — new as of today. `jobs/`
  is its own independent git repo (separate `.git`, own remote), pushed. The
  parent repo's `.gitignore` still excludes `jobs/` so there's no conflict
  between the two repos. `gh` CLI is now installed and authenticated as
  `JasonHunt3r`.
- Doc versions current as of today: `jhg_gcode_hygiene_1_8.md`,
  `jhg_shop_file_standards_1_9.md`, `jhg_troubleshooting_and_build_discipline_1_5.md`,
  plus new `jhg_mazerunr_workflow_1_0.md`. `CLAUDE.md` doc index reflects these.

## 2. Panel C plunge-alignment bug — fixed and verified (Confirmed)

`emit_body_outline()` in `jobs/panel-c/jhg_body_pnlC_rot17.py` arc-fit the
rough/penultimate/finish point lists from array index 0, while each pass's
actual plunge point came from a separately-placed SVG marker — never
reconciled, producing a ~400mm full-depth chord jump on the first cutting move
after every plunge. **This exact fix was already a documented standing rule**
(`ARC-FIT PATH ROTATION`, gcode hygiene doc) that the generator simply wasn't
checked against — re-derived independently before that was discovered.

**Fix applied:** rotate each point list to its own plunge index before
arc-fitting. Verified by diffing the regenerated `.nc` against the prior
output — chord jump confirmed gone from all three passes.

**Also fixed:** the generator's hardcoded input-SVG filename (a stale relative
path) is now a CLI argument: `python jhg_body_pnlC_rot17.py <input_svg>`. It no
longer overwrites its input; it writes two matched outputs sharing the input's
filename stem: `<tag>.nc` and `<tag>_preview.svg`. **This convention has only
been applied to panel-c** — panel-a, panel-e, and both headstocks still have
the old hardcoded-filename behavior.

**Current good file:** `jobs/panel-c/jhg_body_pnlC_rot17_1_6_6.nc` (+ matching
`.py`/`.svg`/`_preview.svg`). This is the file Jason confirmed as the one to
test-cut next. Known-clean on: plunge rotation (fixed), no pocket code path
(structurally confirmed absent), no EMG winding-reversal call (confirmed
absent from `emit_emg_ear()`). **Known-open:** `samples_per_curve=30` in this
file may or may not be sparse enough to trigger a documented concave-arc
defect (see §4) — never confirmed either way against this specific geometry.

## 3. Doc edits made, then reverted (Confirmed — important not to redo blindly)

Two doc edits (an EMG-winding-rule rescope in gcode hygiene, and a new
"VERIFY BY DIFFING NC OUTPUT" section in the troubleshooting doc) were written
based on a claude.ai session ("App Claude") summary of a Panel C incident,
**without verifying against the actual session or files first** — same mistake
the project has been trying to eliminate all day. Jason caught it
("you are jumping the fucking gun") and both edits were reverted. If a later
session (App Claude or otherwise) re-raises this incident with verifiable
specifics (actual session quotes, file evidence), it's worth re-evaluating —
but verify before writing anything into the docs this time.

## 4. Historical audit — what's confirmed vs. still open

Two source docs exist in this folder from a 52-conversation audit of the
pre-Claude-Code project history: `jhg_orphaned_knowledge_1_1.md` and
`jhg_failure_mode_report_1_0.md`. Both are App-Claude-authored summaries —
useful as a lead index, not as ground truth. Key corrections made against real
files during today's session:

- **Panel A's repo copy is stale.** `jobs/panel-a/jhg_body_pnlA_rot17.py` is
  dated 2026-03-22 internally. The actual last-cut file — different,
  architecturally more mature (interleaved pocket+outline per Z-level) — is
  `jhg_body_pnlA_rot17_8_1.py` in `~/Downloads`, dated 2026-04-04. **The repo
  should eventually be updated to carry this file instead**, but that wasn't
  done today (out of scope, flagged only).
- **The "C1/C2 bridging" bug (traversing to the wrong pocket endpoint) does
  NOT reproduce in that real v8_1 file** — hand-traced the interleaved bridge
  logic and it's self-consistent; also regenerated the actual `.nc` in a
  scratch dir and ran an automated jump-detector against it. Clean. This
  contradicts an App Claude "elevated / active bug" verdict from the
  orphaned-knowledge audit — corrected by direct execution, not argument.
- **A real, previously undocumented bug was found in that same file's
  `reverse_commands()`:** it silently drops its last segment when the first
  forward command is an arc spanning more than one sample point (confirmed via
  direct test — fed it a synthetic curve, proved the reversed path stops short
  of the true start point). Checked against the real panel-a geometry: the gap
  is small everywhere in this file (0.3–8.7mm). **Real bug, not yet fixed
  anywhere, but too small to explain the catastrophic jam below.**
- **The actual 2026-04 gantry jam is still unexplained.** Jason clarified
  (correcting an initial misreading — this incident is about **Panel C**, not
  Panel A) that Claude added a neck pocket to Panel C shortly before a final
  cut attempt — a shape that **never had a pocket, ever, in any saved file**.
  The jam was a full-depth straight-line traverse between two coordinates that
  only make sense in Panel A's pocket-navigation logic, grafted into Panel C.
  Searched every locally-saved Panel C `.py` (none contain pocket code, all
  the way through the latest, "Big Ears 6," 2026-04-06) and every locally
  saved Panel C `.nc` (scanned for exactly this signature — a full-depth line
  crossing the real bolt-hole region — using the actual bolt coordinates, not
  a guess). **Nothing found. The specific file where this happened was never
  saved locally** — it likely exists only inside a claude.ai conversation that
  was never exported. **Root cause of the jam remains open.**

## 5. Collaboration lessons reinforced today (Confirmed, saved to memory)

Not project-doc changes — saved to this Claude Code instance's cross-session
memory (`~/.claude/projects/.../memory/`), which a fresh Claude Code instance
on this machine will already have access to without re-deriving:

- `feedback_ask_for_clarity.md` — refined with a concrete/generalized
  instruction distinction (concrete instructions: just execute; generalized
  ones like "bring it up to spec": investigate, then state the read and the
  **criterion** being used to judge correctness — not just the conclusion —
  before executing). Includes the Panel C "make it match Panel A" incident as
  the worked example.
- `feedback_claudecam_process.md` — the standing instruction to verify
  App-Claude-sourced claims against real files before acting on them, with
  today's corrections as the concrete precedent.
- `project_claudecam_setup.md` — updated with the repo split, the plunge-fix,
  the reverted-doc-edit correction, and the still-open jam mystery.

## 6. New architecture direction — shared panel geometry (Reported, designed today, NOT implemented)

This is the live thread to pick back up. Domain model, confirmed by Jason
directly (not App Claude) across several messages today:

**Panel stack:** A (top) / "B" (A's back face — a naming artifact from an
earlier mixup, not a real fourth panel) / C (next layer down) / E (true heel,
no EMG, was next in the work queue when the project stalled in ~April 2026).

**Confirmed relationships:**
- **C and E share an identical outer perimeter AND identical bolt-dip
  positions.** Either file can serve as the master for this. Dips are shallow
  reference marks for hand-drilling, not CNC-bored holes.
- **A's outer perimeter = C's, minus the neck/heel cutout region.** The
  divergence points (where A's path breaks away from C's shared perimeter to
  route around the missing heel) are called **C1/C2** in the existing docs
  and code comments — not yet pinned to actual coordinates for this new
  architecture.
- **A also has its own bridge-position dips**, separate from the shared
  neck-bolt dips — easy to lose during a refactor, explicitly called out by
  Jason as something not to drop.
- **EMG cavities differ in shape between A and C, but partially share
  geometry too:** Panel A's cavity is a plain rectangle (pickup body passes
  straight through the top face there). Panel C's cavity is that same
  rectangle **plus ear-tab extensions** on each long side, which recess the
  pickup's mounting wings so the wood itself is the mount (rear-access
  mounting, no visible bracket) instead of a standard flange-mount pickup
  needing the ears to reach the top. **On each long side, the straight
  rectangle-edge run is literally the same curve as the corresponding edge of
  Panel A's plain rectangle — shared geometry — and only the ear-tab
  detour diverges.** Same "identical-by-construction where shared" principle
  as the outline, just nested inside one feature instead of applying to the
  whole outer boundary.

**The core technical goal:** generate each genuinely-shared curve/edge once,
and have every consuming panel file emit the *literal same* G-code for it —
not "the same function called on separately-parsed geometry," which is what
happens today and has no mechanism forcing agreement between files.

**A real, verified root cause for why "same design curve" can silently drift
between files today:** `CoordTransform` in both `jhg_body_pnlA_rot17.py` and
`jhg_body_pnlC_rot17.py` computes its NC origin independently, as the
*bounding-box center of that file's own body outline point set*. Panel A's
outline includes a pocket detour Panel C's doesn't — even a pixel-identical
outer curve can produce two different computed origins between the two files
purely from that structural difference, before any offsetting or arc-fitting
happens. **Jason's proposed fix: the origin should be tied to a fixed
stock/document size, not derived from either shape's own bounding box** — this
removes the drift mechanism by construction rather than by convention.

**Important context, stated directly by Jason:** this is the same strategic
idea that was behind the incident described in §4 (the Panel C pocket jam) —
but that incident was **not actually an attempt at this idea**; Claude did
something else under its name (per Jason's account, corroborated by nothing
in this generator's saved code path ever showing a proper shared-module
mechanism). So the idea itself is untested, not disproven. Today's process
work (NC-diff-gate thinking, real git history, ask-before-big-changes
discipline) exists specifically to make an idea like this survivable to build.

**Explicitly NOT yet decided / needs Jason's input before implementation
starts, not something to assume:**
1. Real stock dimensions (previously mentioned as 18×18" MDF squares — confirm
   current) and where the fixed origin should sit on that stock.
2. Actual C1/C2 coordinates, pulled from the real files, not estimated.
3. Actual EMG-cavity divergence coordinates (where each long side's shared
   rectangle run ends and the ear-tab detour begins), pulled from real files.
4. Sharing mechanism — a literal shared `.py` module both generators import,
   vs. a generated fragment file each build reads. Leans toward a shared
   module given the shop convention that the `.py` is the deliverable, but
   not decided.
5. Whether E is in scope for this pass or deferred — Jason's last message
   leaned toward doing "all 3 panels" together rather than a smaller test
   piece, reasoning that a toy test case isn't worth inventing when the real
   thing has to be built and cut eventually anyway; validation gate should be
   simulation (CAMotics — named in the shop docs, never actually used per the
   failure-mode report) before anything physical, not a smaller physical test.

**Also open:** `jobs/panel-e/jhg_body_outline_bolts_rot17.py` already exists in
the repo, but its own header says "outline + **bolt reference dips**" which
doesn't match Jason's description of Panel E needing bolt dips too — actually
this might be consistent (re-read: Jason confirmed Panel E DOES get bolt dips,
same positions as C) — but the file's docstring predates today's design
conversation and hasn't been checked against the confirmed domain model.
**Worth reconciling before treating this file as a starting point.**

## 7. Pacing note for whoever picks this up

Jason has been explicit, more than once today, about wanting investigation
and a stated read-plus-criterion **before** execution on anything generalized
or structural — not a pile of clarifying questions, and not silent action on
an assumption either. He's also explicitly not interested in being pushed
toward "next steps" faster than he's asking for them (see §5,
`feedback_claudecam_process.md`, and the corrected memory in
`feedback_ask_for_clarity.md`). The shared-geometry work in §6 is real and
wanted, but treat "what's next" as his call to make, not something to
propose repeatedly.
