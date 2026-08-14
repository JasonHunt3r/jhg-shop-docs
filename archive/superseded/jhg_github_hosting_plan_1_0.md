# JHG Doc Library -- GitHub Hosting and Hyperlinking Plan
Jason Hunter Guitars -- Future Work Plan

---

## The Goal

Host the JHG doc library on GitHub so that:
- Claude can fetch documents directly via URL instead of requiring uploads
- The runbook links out to specific sections of the workflow doc for deeper context
- All docs are version-controlled and always current
- Upload overhead at session start drops to a single short runbook

---

## How It Works

### Fetching Docs from GitHub

Claude has a `web_fetch` tool that retrieves content from any stable URL. GitHub serves raw file content at predictable URLs:

```
https://raw.githubusercontent.com/[username]/[repo]/main/[filename].md
```

A session opener could include: "Fetch the current runbook from [URL]" and Claude reads it directly -- no upload required. The workflow doc, widget spec, and any other reference docs can be fetched on demand when the runbook cross-references them.

### Markdown Hyperlinks

Markdown supports anchor links to headings. Every `##` heading in a GitHub-hosted `.md` file becomes a linkable anchor. Heading text is lowercased, spaces become hyphens, punctuation stripped:

```markdown
[Full bezier correction algorithm](jhg_claudecam_workflow_1_0.md#phase-3----correct-control-points)
```

In the runbook, any section that says "*[Full algorithm: workflow doc → Phase 3]*" becomes a live link to that exact section. Claude can follow it with `web_fetch` if needed.

---

## Proposed Repo Structure

```
jhg-shop-docs/
├── README.md                          ← repo overview, how to use
├── workflow/
│   ├── jhg_claudecam_workflow_1_0.md  ← long version, full methodology
│   ├── jhg_claudecam_runbook_1_0.md   ← short version, operational brief
│   └── jhg_neck_pocket_details.md     ← job-specific reference
├── utilities/
│   ├── jhg_scale_widget_1_0.md        ← widget spec
│   └── jhg_scale_widget_generator.py  ← widget generator (to be built)
├── standards/
│   ├── jhg_gcode_hygiene_1_6.md
│   ├── jhg_shop_file_standards_1_7.md
│   ├── jhg_svg_diagram_workflow_1_4.md ← deprecated, replaced by claudecam docs
│   ├── jhg_troubleshooting_and_build_discipline_1_3.md
│   └── jhg_bezier_offset_toolpath_method_1.md ← deprecated, absorbed into workflow
└── session_starters/
    └── jhg_session_opener.md          ← template for starting a new session
```

---

## Session Starter Template

Once the repo is live, a session opener could look like:

```
Fetch and load: https://raw.githubusercontent.com/[username]/jhg-shop-docs/main/workflow/jhg_claudecam_runbook_1_0.md

We're working on [task description]. The SVG is attached. Proceed with startup protocol.
```

Claude fetches the runbook, reads it, runs the startup protocol on the uploaded SVG. No bulk upload of reference docs required.

For deeper work requiring the full workflow doc, the runbook's cross-reference links point Claude directly to the relevant sections.

---

## Runbook Hyperlink Updates Needed

Once hosted, update these cross-references in `jhg_claudecam_runbook_1_0.md` from plain text to live links:

| Current text | Links to |
|-------------|----------|
| `jhg_claudecam_workflow_1_0.md` → Phase 3 | `#phase-3----correct-control-points` |
| Shop File Standards → SVG-to-G-Code Methodology | `jhg_shop_file_standards_1_7.md#svg-to-g-code-methodology` |
| `jhg_scale_widget_1_0.md` | `utilities/jhg_scale_widget_1_0.md` |
| G-Code Hygiene → G2/G3 Arc Commands | `jhg_gcode_hygiene_1_6.md#g2g3-arc-commands----radius-constraint` |

---

## Steps to Execute

1. **Create GitHub account** (or use existing) and create repo `jhg-shop-docs` (private or public -- private recommended for shop docs)
2. **Upload current doc versions** in the folder structure above
3. **Update runbook cross-references** to live GitHub raw URLs
4. **Add README** explaining the repo structure and how Claude uses it
5. **Test session opener** -- confirm Claude can fetch and read the runbook via URL
6. **Retire upload workflow** for standard sessions once fetch workflow is confirmed

---

## Notes

- Private repos require a GitHub personal access token for raw URL access. Public repos work with no auth.
- Consider whether shop docs should be public (easier for Claude access) or private (protects IP). A public repo with no sensitive parameters is fine -- the actual G-code and SVG files stay local.
- Version control on GitHub means every doc change is tracked. Old versions are always recoverable.
- The `jhg_svg_diagram_workflow` doc series is superseded by the claudecam docs and can be archived rather than maintained going forward.
- `jhg_bezier_offset_toolpath_method_1.md` is fully absorbed into `jhg_claudecam_workflow_1_0.md` and can be archived.
