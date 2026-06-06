# FileMaker Layout XML Skill

This skill gives Claude a deterministic, empirically verified foundation for generating FileMaker layout object XML — the `fmxmlsnippet type="LayoutObjectList"` clipboard format used by FileMaker's Layout mode paste handler.

## What this skill does

When this skill is active, Claude will:

- Generate paste-ready layout XML from plain descriptions ("add a field, a label, and a button to this layout")
- Review existing layout XML for silent-failure risks before you paste it
- Analyse Save-as-XML exports to understand layout structure
- Generate correctly ordered elements, correct flag values, and correct minimal structures — without guessing

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

**With a DDR:**
> Attach your DDR or DDR export and Claude will use real field, layout, and relationship names from your solution.

## Pasting into FileMaker

Layout mode requires the `fmxmlsnippet type="LayoutObjectList"` format on the clipboard in FileMaker's internal clipboard format — not plain text. This skill has been tested with the **MBS Plugin** installed. Plugin-free clipboard conversion options are available in the FileMaker community and should work with this format, but have not been tested by Clockwork.

## What this skill does not cover

- DDR (`LayoutCatalog` / `LayoutObject`) format — that is a different serialisation
- Chart objects (`typeID="CHRT"`) — these contain binary data not reproducible via paste
- Graphic objects — image data is not portable
- Script step library — see the companion FileMaker Script XML Skill
