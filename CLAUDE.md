# JHG ClaudeCAM

Jason Hunter Guitars — CNC toolpath methodology + build archive. This project designs CNC toolpaths as SVG bezier curves (measured and corrected for offset accuracy before any linearization) rather than using conventional CAM auto-generation, then hands off to a Python generator that emits G2/G3 arc-based G-code for a TwoTrees TTC450 PRO.

**Status (2026-08):** The Tenor guitar build (Panel A/C/E body, Headstock Neo 3/4) is parked, not actively being cut. This repo is set up for whenever it resumes.

## Layout

- Root `.md` files — the methodology docs, git-tracked, source of truth. Also hosted at `github.com/JasonHunt3r/jhg-shop-docs` (public — kept public so it stays URL-fetchable, a holdover from the claude.ai-era workflow; raw-URL fetching itself is no longer needed now that these are read locally).
- `transition/` — historical records from the claude.ai → Claude Code migration (2026-08): session handoff, failure mode report, orphaned knowledge extraction, chronology/provenance. Context, not methodology — read only when a question is about project history.
- `archive/superseded/` — old versions of methodology docs and retired schemes, kept browsable on disk. Never read these for current rules; the root copy is always the live one.
- `jobs/` — build artifacts: the current `.py` generator + `.nc` output + verification `.svg` per component, one job per subfolder. Excluded from this repo's `.gitignore` and instead its own **independent git repo**, pushed to the **private** `github.com/JasonHunt3r/jhg-shop-jobs` (2026-08-13). Each subfolder also has a `history/` subfolder for quick pre-overwrite snapshots on top of git. Older pre-migration iterations and the full session-export history still live in `~/Desktop/claude docs/` and `~/Downloads`, not here.

### Simulation — CAMotics (2026-08-15)

Each body-panel job folder carries a `.camotics` project next to its NC, with
the real tool (6.35mm cylindrical, `BIT_R` 3.175) and stock (400 x 430 x 16mm,
origin centred, Z0 at the top) already set. Open one in the CAMotics GUI to
watch the cut, or run it headless:

```
cd jobs/panel-a
DYLD_FALLBACK_LIBRARY_PATH=/Applications/CAMotics.app/Contents/Frameworks \
  /Applications/CAMotics.app/Contents/MacOS/camsim --binary <project>.camotics out.stl
```

The `DYLD_FALLBACK_LIBRARY_PATH` is needed because `camsim` links `libcairo` by
absolute `/usr/local/lib` path, which does not exist on Apple Silicon — the app
bundle ships its own copy in `Frameworks/`. Nothing needs installing. (The
`gcodetool` binary in the same directory needs no workaround and is useful on
its own: `--linearize` expands arcs through an independent interpreter, which
is how the arc emission was cross-checked.)

**What simulation can and cannot settle.** It runs at 0.6mm voxel resolution,
so it confirms topology and gross correctness — the right part, in the right
place, nothing cut outside the stock — and cannot see the 0.1mm-class wall
accuracy `ARC_FIT_TOL` and `PATH_DEV_MAX` govern. It says nothing about feeds,
chip load, MDF behaviour, bed flatness or workholding. A clean simulation is
not a cut, in exactly the way a clean `verify_nc.py` is not a cut.

*Expected and not a bug:* the part never separates. `DEPTH_SCHEDULE` holds 1mm
short of full depth until the bed is planed, so the simulated solid keeps a
floor at Z-15.

### Generator CLI convention (as of the Panel C fix, 2026-08-13)

Job generators take the input SVG as a CLI argument — `python jhg_body_pnlC_rot17.py <input_svg>` — rather than a hardcoded filename. They never overwrite the input; they derive a version tag from the input filename's stem and write two matched outputs: `<tag>.nc` and `<tag>_preview.svg`. Apply this same pattern when touching other job generators (panel-a, panel-e, headstocks) that still have the old hardcoded-filename behavior.

## Doc index

| Doc | Read for |
|---|---|
| `jhg_troubleshooting_and_build_discipline_1_6.md` | Collaboration protocol — read this first, every session. Includes CHECK EXISTING RULES BEFORE DEBUGGING, BUG CLASS RECURRENCE, and the shop/engineering vocabulary table |
| `jhg_claudecam_workflow_1_1.md` | The SVG/CNC workflow — visualization hierarchy, smoothness checks, phases 1-4 (identify → measure → correct → split), bezier correction algorithm, layer/visibility rules, spatial language |
| `jhg_gcode_hygiene_2_0.md` | The G-code authority — machine constants (TTC450 PRO), output rules, feed rates, command selection, arc fitting + derived `ARC_MAX_R` + radius constraint, **PATH FIDELITY** (the error `ARC_FIT_TOL` does not bound), five-stage FINISH PASS SEQUENCE, HELICAL ORBIT hole strategy, ARC-FIT PATH ROTATION, point cleanup |
| `jhg_shop_file_standards_2_0.md` | Cross-file conventions — the `.py` generator is the deliverable, section labeling, PARAMETERS block (canonical values live in generators; docs cite, never restate), parameter risk taxonomy, bezier sample density, offset method (buffer() vs PyClipper, MITER_LIMIT) |
| `jhg_scale_widget_1_0.md` | Scale/orientation widget spec used in every JHG SVG |
| `jhg_neck_pocket_details.md` | Job-specific: neck pocket fit diagram zone codes, `POCKET_EXPAND` sign convention + value history |
| `jhg_mazerunr_workflow_1_0.md` | Design record for the unfinished MazeRunr pocket-clearing subsystem — Ring Stitch v6, adaptive overlap search, open PTW question. Not used by any current job |
| `archive/superseded/jhg_github_hosting_plan_1_0.md` | Historical — the claude.ai-era fetch-by-URL scheme. Superseded by direct file access in Claude Code; kept for reference only |
| `transition/jhg_session_handoff_20260815_2.md` | **Current handoff — start here when resuming.** Repo state, what the second 2026-08-15 session changed (derived `ARC_MAX_R`, the sweep rule, path fidelity, input guards), open items, and traps |
| `transition/jhg_session_handoff_20260815.md` | Earlier the same day — still the reference for the CLOSED Panel C chord history. **Wrong about `ARC_MAX_R` and its open item A**; superseded by the above for anything they both cover |
| `transition/jhg_session_handoff_20260813.md` | Earlier migration-era handoff — superseded by both of the above for anything they cover |
| `transition/jhg_failure_mode_report_1_0.md` | Analysis of Claude-caused defects across the 52 claude.ai conversations (Feb–Apr 2026) |
| `transition/jhg_orphaned_knowledge_1_1.md` | Knowledge extracted from old conversations that never made it into the methodology docs |
| `transition/jhg_chronology_and_provenance_1_0.md` | Reconstructed timeline of the claude.ai era — use for trace-back: where an issue first appeared and what shipped downstream of it |

## Collaboration rules (condensed — full version in the troubleshooting doc)

- **Scope of action = exactly what was asked.** Adding new layers/files is fine; modifying or hiding existing elements without being asked is not.
- **Label findings:** Confirmed (isolated test, proven) / Precaution (hygiene, unverified impact) / Conjecture (untested theory). Never present unlabeled speculation as fact.
- **Flag inherited context** — anything from outside the current session (old handoff docs, memory, past chats) — before acting on it. State what it is and where it came from.
- **Implementation must match the agreed plan.** Before writing a generator or G-code file: re-read the relevant methodology section, verify the plan matches, and if it can't be implemented as documented, stop and say so rather than silently downgrading.
- **File output discipline:** uploaded/external files are read-only — copy before editing, verify writes on disk. Increment version suffixes on reissue (stale filenames get cached).
- **Check existing rules before debugging.** Search the doc library for the mechanism involved before diagnosing a bug from scratch — a bug that looks novel from inside one session is often already a documented, solved problem in a different file.
- **Bug class recurrence.** A bug found and fixed in one pass/section of a generator is a strong prior the same pattern exists elsewhere in the same file. Check every location using that mechanism before declaring the bug class closed, not just the one that was found.

## When resuming SVG/CNC work — ON UPLOAD, DO THIS FIRST

Before doing anything spatial with a JHG SVG:

1. Search for `id` containing `scale-widget`. Read `data-origin-x`, `data-origin-y`, `data-axis-angle`. Confirm against the SCALE block comment. (No widget: use the SCALE block. Neither: stop and ask.)
2. Find the centerline segment, extend it conceptually, establish the object frame.
3. Declare: "Scale widget at [origin], axis angle [N]°. Up in object frame is toward [direction]. Scale confirmed at [value] px/mm." Jason corrects this if wrong before any work proceeds.
4. Search for the cut path (identifier list in the workflow doc, Phase 1). Report what was found.

Then follow `jhg_claudecam_workflow_1_1.md` for the phase pipeline. (The old condensed runbook is retired to `archive/superseded/` — the workflow doc is the single source; its one orphan rule, the lower-confidence identifier qualifier, was moved into Phase 1.)
