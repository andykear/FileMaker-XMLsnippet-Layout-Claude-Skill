# FileMaker Layout XML Skill for Claude

A Claude skill that gives AI models a deterministic, empirically verified foundation for generating and analysing FileMaker Layout mode XML (`fmxmlsnippet type="LayoutObjectList"`).

Developed by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

---

## The problem this solves

FileMaker's Layout mode accepts layout objects via clipboard paste in a specific XML format. Without explicit knowledge of that format, AI models guess — and FileMaker pastes malformed objects silently, with no warning or error.

Effective AI-to-FileMaker workflows require a clear boundary between what AI should determine (the layout logic and content) and what must be deterministic (the XML structure). This skill provides the deterministic layer: a fully verified map of every object type, every element ordering constraint, every flag value, and the paste-handler rules that cause silent failures when violated.

The XML shape is knowable. This spec makes it known.

---

## Keeping AI focused on what it is good at

AI models are generative by nature — they predict, they infer, they improvise. That is exactly what you want when reasoning about what fields belong on a layout and how they should be arranged. It is exactly what you do not want when element order determines whether FileMaker silently drops your objects.

This skill keeps AI focused on what it is good at. The structure is handled deterministically. Claude handles the logic.

---

## How the specification was built

This is not a prompt or a set of guidelines assembled from documentation. FileMaker publishes no formal specification for the `fmxmlsnippet` layout clipboard format.

The specification was built entirely through empirical reverse-engineering: generate XML → paste into Layout mode → save → copy back out → diff against native output. Every object type, every attribute, every element ordering constraint was established through round-trip testing and validated against analysis of 35+ production layouts across 8 real-world applications, totalling millions of lines of exported XML.

Silent failure modes — where FileMaker accepts malformed XML and drops elements without any error — were systematically identified and documented.

The result is a formal specification for a format that Claris has never documented.

---

## What's in the box

```
SKILL.md                                   Claude skill definition
README.md                                  This file
references/
  filemaker_layout_xml_rules.md            Full specification (v1.0, ~900 lines)
```

---

## Specification highlights

- All 18 layout object types documented with minimal generation examples
- Element ordering constraints confirmed via round-trip — order matters and FM is silent about violations
- Object `flags` bits decoded: bit 2 = HideCondition present, bit 14 = sliding, bit 16 = named object, bit 31 = locked
- `FieldObj` flags fully decoded: not-enterable, tab order, Quick Find, calendar button, auto-complete
- `displayType` values confirmed for all control styles: edit box, drop-down list, pop-up menu, checkbox set, radio button set, drop-down calendar
- `pictFormat` values confirmed for all container display modes
- Minimal generation forms verified — `ExtendedAttributes`, `FullCSS`, `DDRInfo`, and `ParagraphStyleVector` confirmed as optional round-trip artifacts
- `TextObj` flags=10 + CDATA encoding confirmed for merge fields
- ButtonBar segment structure: correct flags, bounds offsets, `TextObj flags="2"`
- TabControl: `TabControlObj` requires its own `Styles`; `TitleCalc` comes after `Styles` in `TabPanel`; `TabPanelObj` is a round-trip artifact
- Popover element order confirmed: `Bounds` → `Styles` → `TitleCalc` → `PopoverObj`
- ConditionalFormatting `Item flags` decoded: bits 0/1/2/7 = fill/text/icon/icon-only
- HideCondition `findMode` attribute documented
- WebViewer structure corrected: inner element is `ExternalObj` not `ExternalObjectObj`
- Script step library cross-referenced to companion FileMaker Script XML Skill

---

## Requirements

- Claude (Pro, Team, or Enterprise)
- Skills support enabled in your Claude organisation

Tested with Claude. Model-agnostic by design — the deterministic approach means any capable model with the specification in context should produce reliable output. Claude is the only model Clockwork has verified against production layouts.

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

**With a DDR:**
> Attach your DDR export and Claude will use real field and relationship names from your solution

---

## Pasting into FileMaker

Layout mode requires the `fmxmlsnippet type="LayoutObjectList"` format on the clipboard in FileMaker's internal clipboard format — not plain text. This skill has been tested with the **MBS Plugin** installed. Plugin-free clipboard conversion options are available in the FileMaker community and should work with this format, but have not been tested by Clockwork.

---

## Companion skill

This skill covers layout objects. For FileMaker script generation, see the companion **FileMaker Script XML Skill**, which covers the `fmxmlsnippet type="FMObjectList"` format used by the Script Workspace.

---

## Licence

CC BY 4.0 — free to use, share, and adapt with attribution.

---

## Contributing

Found an object structure that doesn't round-trip? Native export that contradicts the spec? Open an issue or PR. The spec improves through community round-trip testing — that's how it was built.

---

## Version history

| Version | Notes |
|---|---|
| 1.0 | First public release. All 18 object types documented. Full round-trip verification across 35+ production layouts. |
