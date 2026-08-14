# JHG Scale Widget
Jason Hunter Guitars -- Utility Doc
*Specification, placement conventions, and generator for the JHG scale widget. Referenced by [jhg_claudecam_workflow_1_1.md](jhg_claudecam_workflow_1_1.md).*

---

## Purpose

The scale widget serves two functions in every JHG SVG document:

1. **Scale reference** -- a 1" square box with metric and imperial ruler axes lets Claude and Jason verify the document scale at any time without relying on a SCALE block comment alone
2. **Orientation anchor** -- the widget's declared axis angle and directional labels give Claude an unambiguous body frame reference, even in heavily rotated documents

---

## Structure

The widget has three components:

**The origin point** -- where the two ruler axes meet. This is home plate. All spatial references to the widget use this point as the anchor.

**Two perpendicular ruler axes** -- equal-length line segments radiating from the origin at 90° to each other. One runs along the document's intended X axis, one along the Y axis. Both carry tick marks at imperial (every 1/5") and metric (every 10mm) intervals with labels at major ticks.

**The 1" reference box** -- a square whose sides are 1" (72px at 72 DPI / `SVG_SCALE` px at other scales). The box sits **inside the 90° sector defined by the axes** -- like fair territory in a baseball field, with the origin at home plate and the axes as foul lines. The box is never on the outside of the axes.

Inside the box, in small monospace text, are four directional labels oriented to the widget's own axis frame:
- `TOP` -- toward the positive Y axis of the widget (body frame up)
- `BOT` -- toward the negative Y axis
- `L` -- toward the negative X axis
- `R` -- toward the positive X axis

These labels are the authoritative fallback for body frame orientation. Claude reads them directly.

---

## SVG Attributes

The widget group must carry these data attributes for Claude to read programmatically:

```xml
<g id="scale-widget-A"
   data-origin-x="[x]"
   data-origin-y="[y]"
   data-axis-angle="[degrees CCW from SVG vertical]">
```

- `data-origin-x` / `data-origin-y` -- the origin point in SVG coordinates
- `data-axis-angle` -- the rotation of the widget's Y axis relative to SVG vertical (0° = aligned with SVG, 17° = body rotated 17° CCW). This is the single number Claude needs to derive the body frame.

If multiple widgets exist in a document, use `scale-widget-A`, `scale-widget-B`, etc. Each carries its own `data-axis-angle` for its associated feature.

---

## Placement Conventions

**Corner placement (scale only):** Widget origin at or near a document corner. `data-axis-angle` matches the document's overall body frame rotation. Used when the widget is present for scale reference and grid alignment only.

**Centerline placement (scale + orientation anchor):** Widget origin placed on the functional centerline of the primary feature. The widget's Y axis aligns with the centerline direction. This is the preferred placement for JHG body documents -- it makes the widget origin a meaningful physical reference point (e.g., the intersection of the neck centerline with the body boundary).

**Arbitrary placement:** Widget origin at any meaningful reference point. `data-axis-angle` declared explicitly. Claude infers the relationship to the body from context and the declared angle.

**When Claude sees the widget:** If the widget origin falls on or near a significant feature axis (within ~5px), treat it as a centerline placement. If it falls near a document corner or against the bounding box, treat it as corner placement. If the placement is ambiguous, ask once.

---

## Generator Function

The widget is generated programmatically. It is never hand-placed. Full implementation lives in `jhg_scale_widget_generator.py`.

**Function signature:**

```python
def draw_scale_widget(
    origin_x,         # SVG x coordinate of the widget origin (home plate)
    origin_y,         # SVG y coordinate of the widget origin
    axis_angle,       # degrees CCW from SVG vertical for the widget Y axis
    svg_scale,        # px/mm -- used to compute tick spacing and box size
    widget_id="scale-widget-A",  # group id
    ruler_length_in=3.5          # how far each ruler extends from origin, in inches
):
```

**The generator:**
- Computes axis directions from `axis_angle`
- Places the 1" box inside the 90° sector (positive X and positive Y from origin in widget frame)
- Generates tick marks at 1/5" intervals (imperial) and 10mm intervals (metric) on both axes
- Places major tick labels at 1", 2", 3" and 10mm, 20mm, 30mm, etc.
- Places TOP/BOT/L/R labels inside the box, oriented to widget frame
- Embeds `data-origin-x`, `data-origin-y`, `data-axis-angle` on the group element
- Returns the complete SVG group element as a string for insertion into any document

---

## Current Instance

The existing widget in JHG body documents (`jhg_scale_widget.svg`, `id="scale-widget-A"`) was hand-built before this spec existed. It is functional for scale reference. It does not yet carry `data-origin-x/y/axis-angle` attributes or directional labels inside the box.

**To upgrade:** run the generator with the same origin coordinates and `axis_angle=17` (for the 17° CCW body rotation) and replace the existing group. The visual appearance will be equivalent; the machine-readable attributes and directional labels will be added.

---

## Version History

- `1_0` -- initial spec, derived from hand-built widget in existing body documents
