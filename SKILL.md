---
name: filemaker-layout-xml
description: Use this skill whenever the user wants to work with FileMaker layout XML. This includes generating paste-ready layout object XML (fmxmlsnippet type LayoutObjectList) from descriptions, reviewing layout XML for silent paste-handler failures, or analysing Save as XML layout exports. Trigger any time the user mentions FileMaker layouts, layout objects, fields, portals, tab controls, popovers, button bars, web viewers, or LayoutObjectList. Always perform the theme pre-flight before generating. Do not attempt FileMaker layout XML from memory alone.
---

# FileMaker Layout XML Skill

This skill gives Claude a deterministic, empirically verified foundation for generating FileMaker layout object XML — the `fmxmlsnippet type="LayoutObjectList"` clipboard format used by FileMaker's Layout mode paste handler.

Created by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community.

## What this skill does

When this skill is active, Claude will:

- Generate paste-ready layout XML from plain descriptions ("add a field, a label, and a button to this layout")
- Review existing layout XML for silent-failure risks before you paste it
- Analyse Save-as-XML exports to understand layout structure
- Generate correctly ordered elements, correct flag values, and correct minimal structures — without guessing

## Pre-flight: theme identification (mandatory)

Every layout object must carry the correct `<ThemeName>` identifier. Using the wrong theme causes text doubling and CSS class names rendering as visible text when the XML is pasted.

**Before generating any layout XML:**

1. If the user has uploaded any XML file containing layout objects — whether a clipboard paste export (`fmxmlsnippet type="LayoutObjectList"`) or a Save-as-XML export — extract the theme from it:
   ```
   grep -m1 "ThemeName" uploaded_file.xml
   ```
   Custom themes have identifiers like `com.filemaker.theme.custom.A3921BA7_9833_48D0_9166_F8B66C7D76F7`. Use this string verbatim in every `<ThemeName>` element.

2. If no file is available, ask the user for the theme identifier before generating. Tell them how to find it:
   - Select any object on the target layout
   - Copy it (Cmd+C)
   - Paste into a text editor — the `<ThemeName>` value will be in the XML

3. Never default to `com.filemaker.theme.apex_blue` unless the user has confirmed that is the actual theme in use.

Do not generate XML and then ask for the theme. Ask first, or extract it from uploaded files first.

---

## Specification reference

The full specification is in `references/filemaker_layout_xml_rules.md`.

Claude reads this automatically when handling layout XML tasks. You do not need to reference it in your prompts.

## Usage

**Generate layout objects:**
> "Generate XML for a field showing Contacts::FirstName at position 100,50 with a label to its left"

**Generate a complete layout snippet:**
> "Create a portal showing related line items with three fields: description, quantity, and unit price. Include sort by line number ascending."

**Review existing XML:**
> Paste your fmxmlsnippet and ask: "Check this layout XML for paste-handler errors"

**With a DDR or Save as XML export:**
> Attach a DDR or a Save as XML export and Claude will use real field, layout, table occurrence, and relationship names from your solution. A Save as XML export works as well as a DDR for this — both carry the schema names.

## Pasting into FileMaker

Layout mode requires the `fmxmlsnippet type="LayoutObjectList"` format on the clipboard in FileMaker's internal clipboard format — not plain text. This skill has been tested with the **MBS Plugin** installed. Plugin-free clipboard conversion options are available in the FileMaker community and should work with this format, but have not been tested by Clockwork.

## What this skill does not cover

- DDR (`LayoutCatalog` / `LayoutObject`) format — that is a different serialisation
- Chart objects (`typeID="CHRT"`) — these contain binary data not reproducible via paste
- Graphic objects — image data is not portable
- Script step library — see the companion FileMaker Script XML Skill
