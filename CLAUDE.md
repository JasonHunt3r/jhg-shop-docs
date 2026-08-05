# JHG ClaudeCAM

Jason Hunter Guitars — CNC toolpath methodology + build archive. This project designs CNC toolpaths as SVG bezier curves (measured and corrected for offset accuracy before any linearization) rather than using conventional CAM auto-generation, then hands off to a Python generator that emits G2/G3 arc-based G-code for a TwoTrees TTC450 PRO.

**Status (2026-08):** The Tenor guitar build (Panel A/C/E body, Headstock Neo 3/4) is parked, not actively being cut. This repo is set up for whenever it resumes.

## Layout

- Root `.md` files — the methodology docs, git-tracked, source of truth. Also hosted at `github.com/JasonHunt3r/jhg-shop-docs` (public raw-URL fetching is no longer needed now that these are read locally).
- `jobs/` — untracked (`.gitignore`d) build artifacts: the current `.py` generator + `.nc` output + verification `.svg` per component, one job per subfolder. These are the most-recent confirmed snapshot, copied in from `~/Desktop/claude docs/`; older iterations and the full session-export history live there and in `~/Downloads`, not here.

## Doc index

| Doc | Read for |
|---|---|
| `jhg_troubleshooting_and_build_discipline_1_4.md` | Collaboration protocol — read this first, every session |
| `jhg_claudecam_runbook_1_2.md` | Condensed operational brief for SVG/CNC sessions — startup protocol, phases 1-4, spatial language |
| `jhg_claudecam_workflow_1_0.md` | Full workflow — same content as the runbook, unabridged, with the bezier correction algorithm |
| `jhg_gcode_hygiene_1_7.md` | Machine constants (TTC450 PRO), G-code output rules, feed rates, arc radius constraint |
| `jhg_shop_file_standards_1_8.md` | Cross-file conventions — the `.py` generator is the deliverable, section labeling, PARAMETERS block |
| `jhg_scale_widget_1_0.md` | Scale/orientation widget spec used in every JHG SVG |
| `jhg_neck_pocket_details.md` | Job-specific: neck pocket fit diagram zone codes |
| `jhg_github_hosting_plan_1_0.md` | Historical — the claude.ai-era fetch-by-URL scheme. Superseded by direct file access in Claude Code; kept for reference only |

## Collaboration rules (condensed — full version in the troubleshooting doc)

- **Scope of action = exactly what was asked.** Adding new layers/files is fine; modifying or hiding existing elements without being asked is not.
- **Label findings:** Confirmed (isolated test, proven) / Precaution (hygiene, unverified impact) / Conjecture (untested theory). Never present unlabeled speculation as fact.
- **Flag inherited context** — anything from outside the current session (old handoff docs, memory, past chats) — before acting on it. State what it is and where it came from.
- **Implementation must match the agreed plan.** Before writing a generator or G-code file: re-read the relevant methodology section, verify the plan matches, and if it can't be implemented as documented, stop and say so rather than silently downgrading.
- **File output discipline:** uploaded/external files are read-only — copy before editing, verify writes on disk. Increment version suffixes on reissue (stale filenames get cached).

## When resuming SVG/CNC work

Read `jhg_claudecam_runbook_1_2.md` and run its **ON UPLOAD — DO THIS FIRST** protocol against whatever SVG is in play (scale widget, centerline, cut path identification) before doing anything spatial.
