# JHG Neck Pocket Details
Jason Hunter Guitars -- Job Reference Doc
*Job-specific reference for the neck pocket fit diagram and fitting workflow.*

---

## Zone Color Codes

Used in the neck pocket fit reference diagram to identify segments for fit reporting.

| Zone | Segment | Color | Hex |
|------|---------|-------|-----|
| Z1 | Left wall | Cyan | `#3ecfb2` |
| Z2 | Left corner radius | Red | `#e03030` |
| Z3 | Heel arc (large radius) | Amber | `#d4870a` |
| Z4 | Right corner radius | Purple | `#9b30ff` |
| Z5 | Right wall | Hot pink | `#ff2d78` |

---

## Orientation

**TOP** = open end (neck slides in)
**BOTTOM** = heel arc end

**Quadrants:** A=top-left, B=top-right, C=bottom-left, D=bottom-right

---

## Fit Shorthand

Report fit issues using zone + condition: `Z1-TIGHT`, `Z3-LOOSE`, `Z2-BINDING`, etc.

---

## POCKET_EXPAND -- Fit Tuning Parameter

The single most-tuned parameter in the neck pocket generator. Controls how much the pocket toolpath is shifted inward from the design geometry before offsetting.

**Sign convention is counterintuitive -- read this before changing the value.** The value is negated at the call site (`offset_open_curve_inward(pocket_pts_mm, -POCKET_EXPAND)`), so the natural-language direction inverts:

- **Larger** `POCKET_EXPAND` → **looser** neck pocket fit
- **Smaller** `POCKET_EXPAND` → **tighter** fit

**Value history (physical test cuts, MDF):**

| Value | Result |
|---|---|
| 0.40 | Too tight |
| 0.50 | Too loose |
| 0.45 | Interim |
| 0.43 | Cut on Panel 3 |
| 0.42 | Tightening |
| 0.41 | Current as of last recorded test cut |

**Format constraint:** XY coordinates use `.3f` precision, Z uses `.1f`. A value in the 0.4x range is safe for XY math but would not survive being run through Z-coordinate formatting (would truncate) -- keep `POCKET_EXPAND` out of any Z-formatted code path.

**Where this lives day to day:** the direction convention and running value history should be kept in the generator's own code comments, right next to `POCKET_EXPAND`'s definition -- that's where anyone changing the value will actually look. This doc is the durable backup copy; if the two disagree, treat the generator's comment as more current and update this table to match, don't assume this table is right.

---

## Source

Extracted from `jhg_svg_diagram_workflow_1_3.md` (Zone / Spatial Language section).
Diagram Python source: `jhg_neck_pocket_diagram.py`

POCKET_EXPAND history recovered 2026-08-13 from a 52-conversation audit of pre-Claude-Code sessions (`jhg_orphaned_knowledge_1_1.md` in `Transition Support Docs/`).
