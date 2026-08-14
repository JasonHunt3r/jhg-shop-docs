# JHG ClaudeCAM

Jason Hunter Guitars — CNC toolpath methodology + build archive. This project designs CNC toolpaths as SVG bezier curves (measured and corrected for offset accuracy before any linearization) rather than using conventional CAM auto-generation, then hands off to a Python generator that emits G2/G3 arc-based G-code for a TwoTrees TTC450 PRO.

**Status (2026-08):** The Tenor guitar build (Panel A/C/E body, Headstock Neo 3/4) is parked, not actively being cut. This repo is set up for whenever it resumes.

## Layout

- Root `.md` files — the methodology docs, git-tracked, source of truth. Also hosted at `github.com/JasonHunt3r/jhg-shop-docs` (public — kept public so it stays URL-fetchable, a holdover from the claude.ai-era workflow; raw-URL fetching itself is no longer needed now that these are read locally).
- `transition/` — historical records from the claude.ai → Claude Code migration (2026-08): session handoff, failure mode report, orphaned knowledge extraction, chronology/provenance. Context, not methodology — read only when a question is about project history.
- `archive/superseded/` — old versions of methodology docs and retired schemes, kept browsable on disk. Never read these for current rules; the root copy is always the live one.
- `jobs/` — build artifacts: the current `.py` generator + `.nc` output + verification `.svg` per component, one job per subfolder. Excluded from this repo's `.gitignore` and instead its own **independent git repo**, pushed to the **private** `github.com/JasonHunt3r/jhg-shop-jobs` (2026-08-13). Each subfolder also has a `history/` subfolder for quick pre-overwrite snapshots on top of git. Older pre-migration iterations and the full session-export history still live in `~/Desktop/claude docs/` and `~/Downloads`, not here.

### Generator CLI convention (as of the Panel C fix, 2026-08-13)

Job generators take the input SVG as a CLI argument — `python jhg_body_pnlC_rot17.py <input_svg>` — rather than a hardcoded filename. They never overwrite the input; they derive a version tag from the input filename's stem and write two matched outputs: `<tag>.nc` and `<tag>_preview.svg`. Apply this same pattern when touching other job generators (panel-a, panel-e, headstocks) that still have the old hardcoded-filename behavior.

## Doc index

| Doc | Read for |
|---|---|
| `jhg_troubleshooting_and_build_discipline_1_5.md` | Collaboration protocol — read this first, every session. Includes CHECK EXISTING RULES BEFORE DEBUGGING, BUG CLASS RECURRENCE, and the shop/engineering vocabulary table |
| `jhg_claudecam_runbook_1_2.md` | Condensed operational brief for SVG/CNC sessions — startup protocol, phases 1-4, spatial language |
| `jhg_claudecam_workflow_1_0.md` | Full workflow — same content as the runbook, unabridged, with the bezier correction algorithm |
| `jhg_gcode_hygiene_1_8.md` | Machine constants (TTC450 PRO), G-code output rules, feed rates, arc radius constraint, ARC-FIT PATH ROTATION |
| `jhg_shop_file_standards_1_9.md` | Cross-file conventions — the `.py` generator is the deliverable, section labeling, PARAMETERS block, bezier sample density, offset method (buffer() vs PyClipper, MITER_LIMIT) |
| `jhg_scale_widget_1_0.md` | Scale/orientation widget spec used in every JHG SVG |
| `jhg_neck_pocket_details.md` | Job-specific: neck pocket fit diagram zone codes, `POCKET_EXPAND` sign convention + value history |
| `jhg_mazerunr_workflow_1_0.md` | Design record for the unfinished MazeRunr pocket-clearing subsystem — Ring Stitch v6, adaptive overlap search, open PTW question. Not used by any current job |
| `archive/superseded/jhg_github_hosting_plan_1_0.md` | Historical — the claude.ai-era fetch-by-URL scheme. Superseded by direct file access in Claude Code; kept for reference only |
| `transition/jhg_session_handoff_20260813.md` | Migration-era handoff notes — labeled Confirmed/Reported/Open; a communication tool, not ground truth |
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

## When resuming SVG/CNC work

Read `jhg_claudecam_runbook_1_2.md` and run its **ON UPLOAD — DO THIS FIRST** protocol against whatever SVG is in play (scale widget, centerline, cut path identification) before doing anything spatial.
