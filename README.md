# FileMaker Layout XML Skill for Claude

[![Stars](https://img.shields.io/github/stars/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill?style=social)](https://github.com/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill)
![GitHub last commit](https://img.shields.io/github/last-commit/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill)
[![License](https://img.shields.io/badge/license-CC%20BY%204.0-green)](https://creativecommons.org/licenses/by/4.0/)

A Claude skill that gives AI models a deterministic, empirically verified foundation for generating and analysing FileMaker Layout mode XML (`fmxmlsnippet type="LayoutObjectList"`).

Created by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

---

## The problem this solves

FileMaker's Layout mode accepts layout objects via clipboard paste in a specific XML format. Without explicit knowledge of that format, AI models guess — and FileMaker pastes malformed objects silently.

Effective AI-to-FileMaker workflows require a clear boundary between what AI should determine (the layout logic and content) and what must be deterministic (the XML structure). This skill provides that boundary.

The XML shape is knowable. This spec makes it known.

---

## Keeping AI focused on what it is good at

AI models are generative by nature — they predict, they infer, they improvise. That is exactly what you want when reasoning about what fields belong on a layout and how they should be arranged. It is the opposite of what you want when emitting XML element order or flag bit patterns.

This skill keeps AI focused on what it is good at. The structure is handled deterministically. Claude handles the logic.

---

## How the specification was built

This is not a prompt or a set of guidelines assembled from documentation. FileMaker publishes no formal specification for the `fmxmlsnippet` layout clipboard format.

The specification was built entirely through empirical reverse-engineering: generate XML → paste into Layout mode → save → copy back out → diff against native output. Every object type, every flag bit, every element ordering rule confirmed through round-trip testing.

Silent failure modes — where FileMaker accepts malformed XML and drops elements without any error — were systematically identified and documented.

The result is a formal specification for a format that Claris has never documented.

---

## What's in the box

```
SKILL.md                                   Claude skill definition
README.md                                  This file
references/
  filemaker_layout_xml_rules.md            Full specification (v2.0, ~1300 lines)
```

---

## Specification highlights

- All 18 layout object types documented with minimal generation examples
- Element ordering constraints confirmed via round-trip — order matters and FM is silent about violations
- Object `flags` bits decoded: bit 0 = ConditionalFormatting, bit 2 = HideCondition, bit 14 = ToolTip, bit 16 = named object, bit 24 = field access-state marker, bits 28/29/30 = WebDirect rendering tier
- `FieldObj` flags fully decoded: not-enterable, tab order, Quick Find, calendar button, auto-complete
- `displayType` values confirmed for all control styles: edit box, drop-down list, pop-up menu, checkbox set, radio button set, drop-down calendar
- `pictFormat` values confirmed for all container display modes
- Minimal generation forms verified — `ExtendedAttributes`, `FullCSS`, `DDRInfo`, `ParagraphStyleVector` confirmed as optional round-trip artifacts
- `TextObj` flags=10 + CDATA encoding confirmed for merge fields
- ButtonBar segment structure: correct flags, bounds offsets, `TextObj flags="2"`
- TabControl: `TabControlObj` requires its own `Styles`; `TabPanelObj` must be included and carries attributes; element order in `TabPanel` is `Bounds` → `Styles` → `Calculation`
- Popover element order confirmed: `Bounds` → `Styles` → `TitleCalc` → `PopoverObj`
- ConditionalFormatting `Item flags` decoded: bits 0/1/2/7 = fill/text/icon/icon-only
- HideCondition `findMode` attribute documented
- WebViewer structure corrected: inner element is `ExternalObj` not `ExternalObjectObj`
- `ScriptTriggers` documented with all four event types: OnObjectEnter, OnObjectExit, OnObjectModify, OnObjectSave
- `ToolTip` element documented: Calculation-based, supports FM expressions
- `LabelCalc` element documented: dynamic button labels with full FM expression support
- CSS selectors documented: `.self`, `.text`, `.icon`, `.row`, `.row_alt`, `.row_active`, `.button_bar_divider`, `.contents`, `.inner_border`, `.repeat_border`, `.baseline`
- `portalFlags` bit table extended with values 17, 56, 401
- `rotation` units confirmed: tenths of degrees (900 = 90°)
- Theme pre-flight: extract `ThemeName` from any uploaded XML before generating
- Script step library cross-referenced to companion FileMaker Script XML Skill

---

## Requirements

- Claude (Pro, Team, or Enterprise)
- Skills support enabled in your Claude organisation

Tested with Claude. Model-agnostic by design — the deterministic approach means any capable model with the specification in context should produce reliable output. Claude is the only model Clockwork has tested against; others have reported success.

---

## Installation

1. Download the zip from the Releases page
2. Extract — you should have `SKILL.md` and `references/filemaker_layout_xml_rules.md`
3. Upload to your Claude organisation's skills library, preserving the folder structure

---

## Usage

Once the skill is installed, Claude will automatically apply it when you ask for FileMaker layout XML. No special prompt needed.

**Generate layout objects:**
> "Generate XML for a field showing Contacts::FirstName with a label to its left"

**Generate a portal:**
> "Create a portal showing related InvoiceLines with three columns: description, quantity, and unit price"

**Review existing XML:**
> Paste your fmxmlsnippet and ask Claude to check it for paste-handler errors

**With a DDR or Save as XML export:**
> Attach a DDR or a Save as XML export and Claude will use real field, table occurrence, and relationship names from your solution. Either works — both carry the schema.

---

## Pasting into FileMaker

Layout mode requires the `fmxmlsnippet type="LayoutObjectList"` format on the clipboard in FileMaker's internal clipboard format — not plain text. This skill has been tested with the **MBS Plugin** in FileMaker 2024 and 2025.

---

## Known limitation — layout retheming / local CSS removal

Layout retheming is under active development and a primary goal. Reliably rethemeing a whole layout, stripping ad hoc LocalCSS overrides and rebinding objects to their proper named theme style, is not yet recommended for production layouts.
The mechanics work: the whole layout round trips, non matched objects pass through verbatim, matched objects rebind. What is not yet settled is which objects should be treated as a match, and whether object types beyond fields and text behave the same way. Until that is proven across more layouts, treat retheme output as a draft to review, not a paste and trust result.
When it lands it will ship with its own instruction guide. The workflow is involved enough to warrant separate documentation rather than a few usage lines here.
This is the most challenging problem across all the repos and is being worked through deliberately rather than shipped early.

---

## Companion repos

Five open-source resources for the FileMaker/Claris community:

[FileMaker Script XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Claude-Skill) — script steps for the Script Workspace

[FileMaker Layout XML Skill](https://github.com/andykear/FileMaker-XMLsnippet-Layout-Claude-Skill) — layout objects for Layout mode

[FileMaker Field Definitions XML Skill](https://github.com/andykear/FileMaker-XML-field-definitions) — field definitions for Manage Database

[FileMaker XML Inspector](https://github.com/andykear/FileMaker-XML-inspector-open-source) — browser-based Save as XML analyser

[FileMaker XML Scrubber](https://github.com/andykear/FileMaker-XML-scrubber) — redacts credentials before sharing with AI tools

---

## Licence

CC BY 4.0 — free to use, share, and adapt with attribution.

---

## Contributing

Found an object structure that doesn't round-trip? Native export that contradicts the spec? Open an issue or PR and Andrew will investigate.

---

## About

Created by Andrew Kear of [Clockwork Creative Technology](https://www.clockworkct.co.uk), specialising in bespoke FileMaker development, automated artwork systems, and hosted FileMaker solutions.

If you're working on a FileMaker project and need expert help, get in touch.

---

## Version history

| Version | Notes |
|---|---|
| 2.0 | Theming and behavioural model. Added the LocalCSS/CustomStyles/FullCSS serialisation model and four cases, the complete Face character-attribute bitmask, the full script-trigger event table with object-type scoping, button icon embedded-SVG streams, button-bar LabelCalc, the FileMaker 2026 CanEntryCalc access-by-calculation element (generated elements confirmed to enforce), and theme independence proven across two themes. Element-order section refined to match round-trip output (see §21). |
| 1.1 | Extended corpus: 45+ layouts, 10 applications. Added ScriptTriggers, ToolTip, LabelCalc sections. CSS selectors table. portalFlags extended. TabPanelObj corrected (not a round-trip artifact). |
| 1.0 | First public release. All 18 object types documented. Full round-trip verification across 35+ production layouts. |
