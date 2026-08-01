---
name: filemaker-layout-xml
description: Use this skill whenever the user wants to work with FileMaker layout XML. This includes generating paste-ready layout object XML (fmxmlsnippet type LayoutObjectList) from descriptions, reviewing layout XML for silent paste-handler failures, or analysing Save as XML layout exports. Trigger any time the user mentions FileMaker layouts, layout objects, fields, portals, tab controls, popovers, button bars, web viewers, anchoring or autosizing, or LayoutObjectList. Always perform the theme pre-flight before generating, and set object anchoring deliberately whenever an object must resize with the window. Do not attempt FileMaker layout XML from memory alone.
---

# FileMaker Layout XML Skill

Created by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community under CC BY 4.0. Attribution required on any reuse, adaptation or redistribution, including any derived or excerpted work.

Deterministic generation and analysis of `fmxmlsnippet type="LayoutObjectList"`, FileMaker's Layout mode clipboard format. Full specification in `references/filemaker_layout_xml_rules.md`.

Most rules below concern silent failures: the XML pastes without error and is wrong on screen or in behaviour, and no structural audit catches it.

## Pre-flight: theme identification (mandatory)

Every object must carry the target layout's `<ThemeName>` verbatim. The wrong identifier causes text doubling and CSS class names rendering as visible text.

1. If the user supplied any layout XML, clipboard export or Save as XML, extract it: `grep -m1 "ThemeName" file.xml`
2. If not, ask before generating. The user gets it by copying any object from the target layout and pasting into a text editor.
3. Never default to `com.filemaker.theme.apex_blue`. It is a placeholder in the examples only.

Ask first. Do not generate and then ask.

## ExtendedAttributes on every TextObj

Two or more Text objects pasted without an `ExtendedAttributes` block on each `TextObj` causes FileMaker to concatenate their text at paste time. Include a standard block mirroring the object's own `CharacterStyle` on every Text, Button, ButtonBar segment, GroupButton child and PopoverButton, even for a single object.

Dynamic calculated titles on non-front tab panels remain unsafe in a batch: use quoted literals (§11, §31).

Not required on `ExternalObj`; omit it there (§15).

## ButtonBar segment labels go in LabelCalc, not Data

Segment text in `CharacterStyleVector > Data` round-trips byte-perfect and renders blank. Put segment labels in `<LabelCalc>` as the last child of each segment Button, `Data` empty, segment `flags="8"`, empty `ButtonObj buttonFlags="2" iconSize="16"`.

PopoverButtons are the exact inverse: label in `Data`, never generate `LabelCalc` (§9, §9.1, §14).

## Anchoring must be set deliberately

`Object flags` bits 28 to 31 are object anchoring and the only part of that field a generator should author. Mixed polarity: left and top inverted, right and bottom direct.

`flags="0"` is left and top, FileMaker's default, correct for most objects. An object meant to fill a resizable window generated with `0` pastes cleanly, looks right at the design size, and never grows. Use `-1073741824` for all four sides. Sixteen-state table in §2.2.

When analysing a DDR or Save as XML export, anchoring is in `LayoutObject > Options` with all four bits direct and unsigned: `clipboard = ddr XOR 0x30000000` (§2.3).

## Keep a snippet under about 150 KB

Above roughly 150 KB, FileMaker pastes every object and silently discards every `Calculation` element. Objects arrive correctly named, positioned and styled, displaying nothing. There is no per-calculation length limit; the constraint is the whole paste. Split large batches (§15.4).

## Web viewers

`externalFlagSet="32865"` for a viewer hosting an HTML UI. `32781` for plain URL display.

`+64` gates whether `FileMaker.PerformScript()` reaches FileMaker. Without it the `FileMaker` object still exists in the page and calls return normally while going nowhere, so `typeof FileMaker` proves nothing.

`+4` and `+8` draw progress bar and status chrome at a fixed height, which swallows a short viewer entirely: a 46pt viewer showed no page content at all. Omit both on any small viewer and always for an app host.

`name` on the Object element is required for any bridged viewer. Suppress the default border for full-bleed with a `LocalCSS` `border-*-style: none` block (§15.2).

## Named theme styles apply by name

`<CustomStyles><Name>STYLE-NAME</Name></CustomStyles>` plus `ThemeName` binds a user-created theme style, no exemplar object needed. Ask the user for the style name. A name absent from the target theme is dropped silently and the object falls back to base appearance (§25.3).

## Scope

Generation targets the clipboard format only. DDR and Save as XML are a separate serialisation, read for analysis but never generated; layout part styling, backgrounds and theme palettes live there and are not carried by the clipboard (§30). Charts (`typeID="CHRT"`) and graphics carry binary data and are not generatable. Script steps are covered by the companion FileMaker Script XML Skill.
