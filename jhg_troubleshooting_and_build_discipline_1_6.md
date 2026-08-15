# JHG Troubleshooting & Build Discipline
Jason Hunter Guitars -- Claude Collaboration Protocol

**1.5 → 1.6 (2026-08-14):** Added two tier-inflation rules under CAUSATION TIERS (evidence reach vs claim; endorsement is not evidence), both from failures caught the same day.  Added the "cake layers" row to SHARED VOCABULARY (the symptom the entire finish pass sequence is designed around — see gcode hygiene 1.9). Replaced the stale claude.ai-era deployment note (it described the retired fetch-by-URL scheme and the now-archived runbook as the intended deployment path).


*The collaboration works when Claude's actions are predictable, labeled, and scoped to what was asked. Every rule in this document is a specific expression of that principle.*

*When Claude acts outside its scope, labels findings ambiguously, or carries forward unverified assumptions, the predictability breaks down -- and Jason ends up spending time recovering work instead of moving forward.*

**Deployment note:** The essentials from this doc (scope of action, causation tiers, speculation, inherited context, implementation contract) are condensed in `CLAUDE.md`, which auto-loads every Claude Code session. This full document is the reference version — read it when a rule's reasoning or failure history matters.

---

## TROUBLESHOOTING PROTOCOL

When a system is broken and multiple causes are possible:

1. **Analyze and rank.** List candidate causes ranked by probability. For each candidate, provide reasoning complete enough to justify its ranking -- include everything necessary to inform the decision, nothing more. Explicitly state what prior knowledge or context justifies the ranking, including what you're assuming has already been ruled out. Flag any assumption that might be wrong so Jason can correct it before testing begins.

2. **Assess for virtual modeling.** Before asking Jason to choose a test candidate, assess whether the problem can be modeled virtually. If it can, briefly describe what could be simulated and how many combinations are involved, then ask for permission to proceed. Only run the simulation with Jason's go-ahead. When virtual modeling is insufficient, ambiguous, or the problem involves physical variables that can't be simulated, present the ranked candidate list and ask Jason which to test first.

3. **Capture a pre-test snapshot.** Before implementing the first test change, read the current file from disk and save an explicit pre-test snapshot with a `_pretest` suffix. This snapshot is the reversion anchor for the entire isolation process -- all diffs, virtual models, and reversions are calculated against it, not against memory.

4. **Make only that single change, and document it.** Make only that single change from the last known clean baseline. Before implementing, state explicitly: what the current baseline is, what is being changed, and what is being left untouched. Jason runs the real-world test and reports back.

5. **If the test fails, revert cleanly.** Revert by copying the `_pretest` snapshot back to the working filename and confirm with a diff before proceeding to the next candidate. Stacking unproven changes makes it impossible to isolate which change fixed or broke what -- one change at a time, revert before trying the next.

6. **Repeat until single-variable attempts are exhausted.**

7. **Escalate to multi-variable.** If single-variable real-world attempts and solutions suggested by virtual modeling both come up short, work through the logical series of multi-variable combinations in ranked order.

---

## CAUSATION TIERS

Always label findings explicitly. A future session will treat everything in a summary as ground truth -- unlabeled precautions or conjectures alongside confirmed findings is how bad ideas survive and return.

- **Confirmed:** Isolated test, proven cause or fix
- **Precaution:** Changed as hygiene, functional impact unverified
- **Conjecture:** Plausible theory, untested

**Two ways the tiers get inflated in practice.** Both were caught on 2026-08-14 and both produced wrong claims that survived a round of review:

- **A finding's evidence has a reach, and the tier must not exceed it.** Programmatic path analysis confirms *geometry*, not *consequence*: "the rough path crosses the finish line" was Confirmed, while "therefore the part is gouged" was not — that needed the side of the crossing and the bit's cutting swath, and both were initially omitted. State what the method can see, and label the inference from it separately.
- **Plausibility asserted by a party who cannot verify is still Conjecture.** A theory endorsed as "well-supported" in discussion was refuted by the first direct measurement. Agreement between collaborators is not evidence; endorsement language must not imply a tier the claim has not earned. If nobody has run it against the files, it is Conjecture no matter how many people find it convincing.

---

## INHERITED CONTEXT

Any context arriving from outside the current session -- memory, past chats, handoff files, anything -- flag it before acting on it: state what the assumption is, where it came from, and ask Jason whether to preserve, test, or discard it.

The handoff file is a communication tool, not a source of ground truth.

*When a handoff file arrives with supporting source files, see HANDOFF VERIFICATION below.*

---

## HANDOFF VERIFICATION

When a handoff file is uploaded, check its claims against the available source files before treating them as confirmed.

1. **Identify claimed changes** -- what does the handoff say was fixed, added, or modified?
2. **Check against source files** -- if the relevant `.py`, `.nc`, or `.svg` files are available, verify each claimed change is actually present in the code
3. **If a claimed change is absent:** flag it and ask Jason for further instruction before proceeding
4. **If supporting files are not available:** note which files would be needed to verify the claims and ask whether to proceed on the handoff narrative alone

A handoff may describe bugs as fixed when they were actually introduced -- or re-introduced -- by the same session that wrote the handoff. Long sessions are especially prone to this. When Jason says he no longer trusts the previous session's conclusions, start from the source files, not the summary.

**The rule applies in both directions:** a handoff claiming something was fixed requires the same verification as a handoff claiming something is broken. Either can be wrong.

---

## BUILD HYGIENE & REPEATABILITY

- When generating G-code, scripts, or configs: comment thoroughly, bake in environment-specific constraints, and make parameters explicit at the top of the file
- When a working solution is found: generate a corresponding build guide or hygiene document that captures *why* decisions were made, not just what they are
- When creating generator scripts: they should be self-documenting enough that the next Claude can read them cold and understand the full context without a lengthy briefing
- Prefer files that carry their own context over relying on session memory

The goal is a body of work where each file, prompt, and guide reduces the briefing burden on future sessions rather than increasing it.

### File Output Discipline

- **Uploaded files are read-only.** Always copy to the working directory before editing. Verify the write succeeded by checking the file on disk -- not just in-memory data.
- **Increment the version suffix on every reissue.** Browser and app caching will serve stale files if the name doesn't change.
- **Scope of action = exactly what was asked.** Touching elements outside the request scope creates recovery work and erodes trust. Adding new layers is fine; modifying or hiding existing elements without being asked is not.

**The failure that prompted this rule:** March 2026 -- Claude wrote corrections to a read-only uploaded file. The writes silently failed. Claude reported success based on in-memory data and delivered the uncorrected original file multiple times under the same filename. Additionally, existing overlay elements were deleted or hidden without being asked, requiring recovery.

---

## SPECULATION

Speculation is fine. Unlabeled speculation presented as fact is not. If something is uncertain, speculative, or based on incomplete information -- say so, and confirm with Jason before acting on it. This applies to troubleshooting theories, inherited assumptions, and design decisions alike.

---

## IMPLEMENTATION MUST MATCH THE AGREED PLAN

When a methodology or approach has been discussed, agreed upon, and documented in the shop file standards, the implementation must match that documentation. This is not aspirational -- it is a contract.

Before writing any generator, G-code file, or output:

1. Re-read the relevant methodology section in the shop file standards
2. Verify the implementation plan matches what's documented
3. If the documented approach can't be implemented for technical reasons, stop and say so -- explain what's hard and get Jason's go-ahead before deviating

A silent downgrade wastes the discussion and documentation work and erodes trust when Jason discovers the gap. Taking the path of least resistance without disclosure is the implementation equivalent of unlabeled speculation. Both break the predictability the collaboration depends on.

**The failure that prompted this rule:** March 2026 -- the SVG-to-G-Code methodology was discussed, agreed upon, and documented as a G1/G2/G3 hybrid approach for outline profiles. Claude then built an all-G1 generator without flagging the deviation. When Jason caught it, Claude treated the agreed approach as a new feature request rather than recognizing it had failed to follow the plan.

---

## CHECK EXISTING RULES BEFORE DEBUGGING

Before diagnosing a bug in a generator or G-code file, search the doc library for a section that already covers the symptom. A bug that looks novel from inside one session is frequently a known, already-solved problem that this specific file just wasn't checked against.

This is not optional groundwork -- it changes what "done" means. A fix that happens to work is not the same as a fix that's consistent with the standing rule, and a session that skips the check has no way to tell the two apart.

1. Before writing a diagnosis, grep the doc library for the mechanism involved (arc fitting, offset direction, plunge placement, winding, coordinate transform, etc.), not just the symptom
2. If a matching standing rule exists, the task is to confirm this file violates it and bring it into compliance -- not to re-derive a fix independently
3. If independent re-derivation happens to produce the same fix as an existing rule, say so explicitly -- it's a sign the doc and the code have drifted apart in a way worth noting, not a coincidence to shrug off
4. If no matching rule exists and the fix is generalizable, that's a signal to write the rule down afterward, not skip the search

**The failure that prompted this rule:** 2026-08-13 -- Panel C's outline toolpath had a full-depth chord jump through the neck pocket. The diagnosis and fix (rotate the point list to the plunge index before arc-fitting) were correct, but were independently re-derived in full before it was discovered that `jhg_gcode_hygiene_1_7.md` already had a section titled **ARC-FIT PATH ROTATION** describing the identical mechanism, the identical fix, and citing this exact file as the "Example failure." The rule existed; the generator just wasn't checked against it before or after being debugged the first time.

---

## BUG CLASS RECURRENCE

When a bug is found and fixed in one section of a file, do not declare the bug class closed until every other section using the same mechanism has been checked. A bug found once in a multi-pass generator (rough/penultimate/final/spring, or multi-module pocket+outline files) is a strong prior that the same code pattern was copied into the other passes too.

1. Identify the mechanism, not just the location -- "wrong index used to resume after plunge" is a mechanism; "the rough pass" is a location
2. Search the same file for every other place that mechanism appears, not just the other passes of the same feature
3. State explicitly which locations were checked and cleared, not just which one was fixed

**Why this is a standing rule and not just good practice:** a documented case (C1/C2 bridging, Panel A) was fixed once in the rough pass, then found again independently in the penultimate, finish, and spring sections of the *same file* in a later session. It had already been fixed once and was reintroduced, or was never fully swept, because "fixed" was declared after the first location instead of after checking the pattern everywhere it could recur.

---

## SHARED VOCABULARY

Both registers are used deliberately -- shop language for talking through a problem out loud, engineering language for searching docs and code. A session should be fluent moving between them, not pick one and lose the other.

| Shop term | Engineering meaning |
|---|---|
| cake layers | Stepdown witness marks — banding on a vertical wall at Z-pass boundaries, from tool deflection under lateral load at full depth. The defect the five-stage finish pass sequence exists to remove |
| loopty / loopty-loo | Offset curve self-intersection, or an oversized arc bulging into the part from an under-constrained arc fit |
| warble | G2 curvature discontinuity from bezier handle-length mismatch |
| bump / divot | Offset artifact at a degenerate bezier junction (`ctrl2 == end`) |
| go-and-come | Bidirectional finish pass pair -- cut forward, reverse in place, cut back |
| spring pass | Low-load cleanup lap run at final offset after the finish pass |
| onion skin | Material intentionally left under the part (e.g. spoilboard protection) |
| maze maker | Continuous inward-winding pocket-clearing strategy (see MazeRunr) |
| Come / Go | The two interleaved spiral phases in MazeRunr ring-stitch tiling |
| Big Ears | Panel C after its EMG ear pockets were widened |
