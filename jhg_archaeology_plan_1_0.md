# JHG ClaudeCAM — Archaeology Plan v1.0

*Scoping document for a fresh session. Written 2026-08-14, after the docs rewrite (commits `99625b4`–`1c252f2`) landed.*

---

## Purpose

Build a **reference layer** recording how the ClaudeCAM pipeline came to be what it is, so future sessions can answer "why is this like this?" and "have we hit this before?" without a deep dive through conversation history.

**This is not operational documentation.** The live doc set (seven files, one home per rule) is current and sufficient to cut wood. Nothing in this plan blocks a template, and nothing produced here should be read during a working session.

### The organizing distinction

| | Live docs | Archaeology layer |
|---|---|---|
| Contains | **Rules** — standing, always true | **Events** — dated, happened once |
| Has a date | No | Yes, always |
| Read when | Doing the work | Asking why, or hitting something familiar |

Keeping events out of the rule docs is what stops this layer from becoming a fourth copy of the methodology. That duplication is the exact problem the August rewrite was spent undoing — do not reintroduce it.

---

## What already exists — do not rebuild

| Artifact | Location | Covers |
|---|---|---|
| `PARAM_CENSUS.md` | project files | Full parameter value history, 57 generators, dated. **This is the parameter archaeology, already done.** |
| `MANIFEST.md` | project files | Every surviving file, dated, by component |
| `jhg_failure_mode_report_1_0.md` | repo `transition/` | Failure categories and structural causes |
| `jhg_chronology_and_provenance_1_0.md` | repo `transition/` | Session-level chronology |
| `jhg_orphaned_knowledge_1_1.md` | repo `transition/` | Open items and unabsorbed knowledge |
| "The failure that prompted this rule" blocks | hygiene 1_9, troubleshooting 1_6 | Individual incidents, indexed *by rule* |

The gap is not raw material. It is **findability** — most of this is organized by rule or by file, and a future dig arrives with a symptom or a question.

---

## Deliverables, in priority order

### 1. Abandoned work register — **start here**

Work that was built, functioned or nearly functioned, and never reached a repo.

Most perishable item on this list. These files exist only in the Evolution archive and the conversation record, and **nothing on disk distinguishes "abandoned" from "superseded."** Every month makes this harder to reconstruct.

Known entries to seed from:

- **`jhg_body_pnlC_rot17_1.py` lineage** (`_1` → `_1_1` → `_1_2` → `_1_3` → `_1_4` in *Big Ears 6 ƒ*, 04-05 to 04-06). Stepped penultimate Panel C, fully wired through emit, overlay, and section headers. Session ended with an unresolved arc-flip loop at the treble-side ear tip. Never centralized. Whether *Big Ears 6* resolved the loop is **open**.
- **Panel C 2.0/0.5 parameter change** (session *Examining Panel C and updating standards*). Approved, applied to a container copy, never downloaded. The doc update from the same session *did* ship — which is why standards carried `PENULT_LEAVE = 0.5` for months with no code behind it.
- Anything else surfaced by the sweep below.

Per entry: what it was, when, why it stopped, where the file is, and whether anything in it is worth recovering.

### 2. Failure index — by symptom

Highest-value single artifact. Mostly reorganization of text that already exists; the missing piece is the **lookup direction**. Current failure records are indexed by rule; a future session arrives with a symptom.

Index on the observable, then point to the rule and the session:

| Symptom | Rule | Where |
|---|---|---|
| Diagonal witness mark across a pocket interior | Go-and-Come Polygon Reversal — Side Index | hygiene 1_9 |
| Machine spins, no XY movement, spindle running | G2/G3 Radius Constraint (GRBL error 33) | hygiene 1_9 |
| Straight chord across the part after plunge | Arc-Fit Path Rotation | hygiene 1_9 |
| Flat facet at a horn tip on the return pass | Go-and-Come Arc Reversal | hygiene 1_9 |
| Frayed exit spike at neck pocket | Interleaved Rough — Stay at Depth | hygiene 1_9 |
| Curve visibly flattened, real points missing | Point Cleanup Rules | hygiene 1_9 |
| Arc cuts across the curve between sample points | Arc Fitting Constraints | hygiene 1_9 |
| Loop at a curve tip / ear tip | *(C1 discontinuity, arc direction flip — no rule yet)* | Panel C sessions |
| Chatter on one long side of a rectangular pocket only | *(winding reversal → one side always climb — unresolved)* | Panel C sessions |
| Banding on a vertical wall | Finish Pass Sequence (cake layers) | hygiene 1_9 |

The last two rows have no rule. Recording them as open symptoms is the point — an unsolved symptom that is *findable* is worth more than one that is not.

### 3. Module notes — one file per module

Description, origin, and rationale for each reusable function in the pipeline. `MODULE REGISTRY` already exists in standards 2.0 as scaffolding.

Per module: what it does, what it replaced, what was tried and abandoned, which failure produced each guard clause, and — most useful — **which generators currently use it.**

That last field is load-bearing. `reverse_commands()` was ported from Panel A to Panel C during the March "revised" session; per CC's lineage check the working Panel C may not have it. A "used by" list makes that class of gap visible without running a census.

Suggested first module: **`reverse_commands()`** — clean origin, documented prompting failure (March horn-tip flattening), a port history between panels, and a live open question about which files have it.

Keep entries rough. Rough and complete beats polished and partial.

### 4. Decision log — optional, only if it stays cheap

Choices with reasoning that isn't obvious from the result: 17° body rotation, `JT_MITER` with `MITER_LIMIT=20` over `JT_ROUND`, conventional vs climb by material, arc-fit before/after rotation ordering.

Cut this entirely if items 1–3 run long.

---

## Method

### The one rule that matters

**Conversation search shows what was decided and drafted. Files show what shipped.** These are different, and conflating them produced two errors in the session that generated this plan — a parameter value and a doc quotation, both traced from sessions and both wrong about shipped state.

So: an App-side finding is a **lead**, not a fact, until CC confirms it against a file.

### But do not gate on verification

Tempting and wrong for this layer. The unverified material is often the most valuable thing in it — "built, worked, never shipped" is exactly what a future dig wants. A verification gate would delete the Panel C stepped lineage from the record, the single most useful find of the August work.

**Label instead of filtering.** Use the existing causation tiers on every entry:

- **Confirmed** — verified against a file on disk or in the archive
- **Precaution** — consistent with the record, not directly verified
- **Conjecture** — inferred from conversation only

An entry marked Conjecture is fine. An unlabeled entry is not.

### Corrections carried forward from the August session

Three findings that a fresh session would otherwise re-derive incorrectly:

1. **The value history is not monotonic generations.** The `2.0/0.5` ladder was adopted Mar 23, ran four days, and was **deliberately rolled back** Mar 27 to `1.0/0.3` with `DIP_DEPTH 1.25→1.0` alongside it. When `2.0` returned Apr 4, `0.5` did not. Do not describe this as drift or as lost work — it was a reversal.
2. **Sedimentation is real but narrow.** It explains Neo 3 (untouched since Mar 19) versus Neo 4 (Gen 1 since Mar 22). It does **not** explain the fleet spread generally. An earlier framing that generators "freeze at their last good session" was refuted by the census.
3. **Two Panel C lineages share a filename.** `Proven Cuts/Panel C proven/jhg_body_pnlC_rot17_1.py` (2026-03-25) and the *Big Ears* `jhg_body_pnlC_rot17_1.py` (2026-04-05) are different files. One carries a known-good stamp, the other is the most evolved. Never treat "the Panel C generator" as singular.

### Division of labor

| App Claude | Claude Code |
|---|---|
| Mines conversation history for events, motives, and who decided what | Verifies claims against files; greps the archive; runs censuses |
| Drafts and organizes the reference files | Confirms file existence, dates, and contents |
| Produces leads with dates and provenance | Promotes leads to **Confirmed**, or flags them as not found |

App cannot read `~/Projects/ClaudeCAM/Evolution/`. CC cannot read the conversation record. Neither half is sufficient alone — that asymmetry is the whole reason this plan splits the work.

---

## Session setup

Load into the project:

1. `MANIFEST.md` *(already present)*
2. `PARAM_CENSUS.md` *(already present)*
3. This plan

Everything else is re-derivable. Do **not** pre-load the live docs — they are current, they are not the subject, and loading them invites re-litigating settled decisions.

### Suggested order

1. Agree the scope of item 1 and the entry format
2. Sweep conversation history for abandoned work; produce a candidate list with dates
3. CC checks each candidate against the archive; label tiers
4. Write the abandoned-work register
5. Reassess before starting item 2 — the failure index may want its own session

---

## Stopping rule

One test, applied ruthlessly: **would anyone actually look this up?**

If not, do not write it. This layer exists to make the next dig shallow, not to be complete. An archaeology project with no stopping rule becomes permanent, and the working set is already done — this is optional work, and it should be able to stop at any point and still be useful.

---

## Open questions to settle in-session

- Do module notes live in the docs repo (readable by App) or the jobs repo (private, code-adjacent)? Bears on whether App can help maintain them.
- Does the failure index live in the live doc set as a navigational aid, or in the archaeology layer? It is the one deliverable here with plausible operational value.
- Repo visibility, unresolved from the August session: public is currently load-bearing for App-side reading, but that is a plumbing consequence rather than a decision.
