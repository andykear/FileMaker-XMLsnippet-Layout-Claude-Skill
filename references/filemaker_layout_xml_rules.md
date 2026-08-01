# FileMaker Layout XML Spec

Created by Andrew Kear of Clockwork Creative Technology and shared openly with the FileMaker/Claris community under CC BY 4.0. Attribution required on any reuse, adaptation or redistribution, including any derived or excerpted work.

Paste-ready FileMaker layout object XML (`fmxmlsnippet type="LayoutObjectList"`), empirically derived from round-trip testing across multiple applications and themes.

**✓** = round-trip verified  **◎** = observed, or verified in one scope and expected to generalise  **○** = unverified: single observation, hypothesis, or survives round-trip with behaviour unconfirmed

**Methodology note:** ✓ certifies round-trip *survival*. Survival does not guarantee *rendering* — FM 26 will faithfully round-trip elements its renderer no longer reads (see §9.1). Where a claim concerns on-screen behaviour, visual confirmation is stated explicitly.

---

## §0 Pre-flight: theme identification

**Every object's `<ThemeName>` element must match the target layout's theme exactly.** Using the wrong theme identifier causes text doubling and CSS class names to render as visible text on paste.

### Finding the correct ThemeName

**From an uploaded XML file — either format:**

The user may upload XML in one of two ways:
- **Clipboard paste export** — objects copied from a FileMaker layout and saved as XML (the `fmxmlsnippet type="LayoutObjectList"` format). This is the most common case.
- **Save-as-XML export** — a full layout export from File > Save a Copy As > XML.

Both contain `<ThemeName>`. Extract it the same way:
```
grep -m1 "ThemeName" filename.xml
```

Custom themes have identifiers like:
```
com.filemaker.theme.custom.A3921BA7_9833_48D0_9166_F8B66C7D76F7
```

**Without any uploaded file:** Select any object on the target layout → Copy → paste the clipboard contents into a text editor → find the `<ThemeName>` value.

### Rules

- If the user uploads any XML file containing layout objects, extract `ThemeName` from it before generating anything. ✓
- If no file is provided, ask for the identifier before generating — not after.
- Never default to `com.filemaker.theme.apex_blue` unless confirmed. It is a placeholder in the examples only.
- Use the extracted identifier verbatim in every `<ThemeName>` element throughout the generated XML.

### Clipboard behaviour: single-object copy

**A single selected Text object copies to the system clipboard as plain text, not as `fmxmlsnippet` XML.** Two or more selected objects — of any type, including two Text objects together — copy correctly as layout XML. This affects any workflow, including the Inspector's own capture routines, that asks a user to "select one object and copy": a lone Text object silently produces no usable XML, with no error shown. When isolating a single Text object for analysis, select it alongside any other object (or a throwaway shape) and discard the extra object from the resulting snippet afterward. ✓

---

## §1 Wrapper

```xml
<?xml version="1.0" encoding="UTF-8"?>
<fmxmlsnippet type="LayoutObjectList">
  <Layout enclosingRectTop="0" enclosingRectLeft="0"
          enclosingRectBottom="100" enclosingRectRight="300">
    <!-- Object elements -->
  </Layout>
</fmxmlsnippet>
```

- `type="LayoutObjectList"` not `FMObjectList` ✓
- `enclosingRect` is metadata — FM ignores it for positioning ✓
- 2-space indent, UTF-8 ✓
- `Bounds` values are floating-point 7dp. Integers are valid — FM normalises ✓

---

## §2 Object element

```xml
<Object type="Field" key="1" LabelKey="0" flags="0" rotation="0">
```

| Attribute | Notes |
|---|---|
| `type` | See §3 |
| `key` | FM reassigns on paste — any integer, duplicates safe ✓ |
| `LabelKey` | Key of associated label object. `0` = no label ✓ |
| `flags` | **Use `0` for generation, except anchoring.** Bits 28-31 are object anchoring and are settable. See §2.1 and §2.2 |
| `rotation` | Tenths of degrees. `0` = no rotation, `900` = 90°, `1800` = 180°. **Text and Button objects only** — confirmed round-trip, both values render correctly rotated. Confirmed NOT functional on shapes: a `Rect` generated with `rotation="900"` pasted with no visible rotation and `FullCSS` came back with `-fm-rotation: 0` regardless of the value set. Do not rely on `rotation` for shapes; it is present on every `Object` element but only Text/Button honour it. ✓ |
| `name` | Optional. The layout object name. Round-trips on Button, ButtonBar, individual bar segments, and WebViewers ✓. FM does not set flags bit 16 on paste-generated named objects — the attribute alone is sufficient. |

### §2.1 Object flags — generation rule

**Use `flags="0"` for all generated objects except when setting anchoring.** Bits 0 to 25 are set by FileMaker from object state and must not be written. Bits 28 to 31 are object anchoring: they are both readable and settable, and are the one part of this field a generator may legitimately author. `flags="0"` decodes to the FileMaker default of left and top anchored, so the zero default remains correct for any object that does not need to resize. See §2.2.

| Bit | Value | Meaning |
|---|---|---|
| 0 | 1 | Has `ConditionalFormatting` ✓ |
| 2 | 4 | Object has a HideCondition ✓ |
| 8 | 256 | **Field object: participates in Find requests ("Apply in Find Mode").** Confirmed on four Field objects across all four field-entry access states, additive and independent of every other flag ✓. Not icon presence — an icon-bearing standalone Button captured directly returned `flags="0"` at the Object level regardless of its icon. Icon presence lives entirely inside `ButtonObj` (displayType + FNAM/GLPH/SVG streams, see §19.3), never in Object flags. |
| 12 | 4096 | Line/Rect: round-trips intact but no visible effect found anywhere in the FM Pro 26 Inspector — treat as an inert/legacy marker; do not generate expecting behaviour ✓ |
| 13 | 8192 | Line/Rect: same as bit 12 — inert on round-trip in FM Pro 26 ✓ |
| 14 | 16384 | Has `ToolTip` ✓. (Placeholder presence is a separate `FieldObj` flag, bit 17, not this Object bit.) |
| 24 | 16777216 | Field access-state marker — Browse mode, part of the full decode in §5.2 ✓ |
| 25 | 33554432 | Field access-state marker — Find mode, part of the full decode in §5.2 ✓ |
| 28 | 268435456 | Anchoring: **left anchor OFF**. Inverted sense. See §2.2 ✓ |
| 29 | 536870912 | Anchoring: **top anchor OFF**. Inverted sense. See §2.2 ✓ |
| 30 | 1073741824 | Anchoring: **right anchor ON**. See §2.2 ✓ |
| 31 | -2147483648 | Anchoring: **bottom anchor ON**. See §2.2 ✓ |

The generation rule for bits 0 to 25 is simple: use `flags="0"` and let FileMaker set these. Bits 28 to 31 are the exception and are covered in §2.2.

**Do not generate bits 3 or 16.** A portal field with the row option engaged, and a natively-named object, both return `flags="0"`. Object naming lives entirely in the `name` attribute; no flag bit is associated with it on any object type. ✓

**Bits 1 and 9 occur on FM 26 output with no known cause.** Observed on consecutive `ExternalObject` (WEBV) captures placed identically: `flags="0"`, `flags="2"` and `flags="512"` across three objects, none set deliberately. Do not generate either; do not treat their presence as meaningful when analysing. ○

**Never generate `260`, `261`, `256`, `65544` or `65545`.** FM 26 encodes no ButtonBar segment state in Object flags at all (§9.1). ✓

---

### §2.2 Object anchoring (autosizing)

Object anchoring is encoded in `Object flags` bits 28 to 31. The polarity is mixed: **left and top are stored inverted**, **right and bottom are stored directly**. An object with default anchoring therefore emits `flags="0"`.

| Bit | Value | Meaning | Sense |
|---|---|---|---|
| 28 | 268435456 | Left anchor **off** | inverted |
| 29 | 536870912 | Top anchor **off** | inverted |
| 30 | 1073741824 | Right anchor **on** | direct |
| 31 | -2147483648 | Bottom anchor **on** | direct |

Values are signed 32-bit. Any combination including bit 31 serialises negative.

#### Anchor values

All sixteen states. Nine were captured by round-trip copy on WebViewer objects, FileMaker Pro 26, macOS, including three predicted before capture, and confirmed in the generation direction; the rest follow from four independent bits. ✓

Verified across object types: Text, WebViewer and Rectangle objects generated at all eight distinct anchor states each landed with exactly the intended anchoring, confirmed against the DDR export of the same layout (§2.3). The decode is object-type independent. ✓

| Anchors | `flags` |
|---|---|
| left + top (FileMaker default) | `0` |
| none | `805306368` |
| top | `268435456` |
| left | `536870912` |
| right | `1879048192` |
| bottom | `-1342177280` |
| top + right | `1342177280` |
| left + right | `1610612736` |
| left + bottom | `-1610612736` |
| top + bottom | `-1879048192` |
| right + bottom | `-268435456` |
| left + top + right | `1073741824` |
| left + top + bottom | `-2147483648` |
| left + right + bottom | `-536870912` |
| top + right + bottom | `-805306368` |
| left + top + right + bottom | `-1073741824` |

#### Generation rule

Emit `flags="0"` unless the object must resize with the window. `0` is left and top anchored, FileMaker's default for a newly placed object.

Set the bits deliberately for anything that must stretch or reposition. Use `-1073741824` for a full-bleed object such as a web viewer hosting an integrated HTML UI (§15).

**An object generated with `flags="0"` and intended to fill a resizable window pastes, renders correctly at the design size, and then does not grow.** Nothing errors, nothing is dropped, and no structural audit catches it.

#### Not portable to the DDR format

DDR and Save as XML use the same four bits with different polarity. See §2.3 for the conversion.

---

### §2.3 Anchoring in DDR / Save as XML

`LayoutObject > Options` carries the same four anchor bits as the clipboard `Object flags`, but **all four are direct** and the value is unsigned.

| Bit | Value | DDR meaning |
|---|---|---|
| 28 | 268435456 | Left anchor **on** |
| 29 | 536870912 | Top anchor **on** |
| 30 | 1073741824 | Right anchor **on** |
| 31 | 2147483648 | Bottom anchor **on** |

No anchors is `0`; all four is `4026531840`. The clipboard's inverted left and top bits are a clipboard-only convention.

#### Conversion

```
clipboard_flags = ddr_options XOR 0x30000000   (then read as signed 32-bit)
ddr_options     = clipboard_flags XOR 0x30000000
```

| Anchors | DDR `Options` | Clipboard `flags` |
|---|---|---|
| none | `0` | `805306368` |
| left + top | `805306368` | `0` |
| top | `536870912` | `268435456` |
| left | `268435456` | `536870912` |
| right | `1073741824` | `1879048192` |
| bottom | `2147483648` | `-1342177280` |
| left + top + right | `1879048192` | `1073741824` |
| all four | `4026531840` | `-1073741824` |

#### Verification

Eight anchor states generated as clipboard XML across three object types (Text, WebViewer, Rectangle), pasted, then read back from a Save as XML export of the same file. All twenty-four objects matched, and the eight paired values satisfy the XOR relationship exactly. FileMaker Pro 26.0.1.51, macOS. ✓

Anchoring is therefore readable directly from a DDR, which matters for whole-solution analysis where no clipboard round-trip is available. `Options` also appears on `Part > Definition` with an unrelated meaning; only `LayoutObject > Options` carries anchoring.

---

## §3 Object types

| `type` | Inner element | Notes |
|---|---|---|
| `Field` | `FieldObj` | |
| `Text` | `TextObj` | |
| `Button` | `ButtonObj` | |
| `ButtonBar` | `ButtonBarObj` | |
| `GroupButton` | `GroupButtonObj` | |
| `Portal` | `PortalObj` | |
| `Line` | *(none)* | Requires `RenderFormat` ✓ |
| `Rect` | *(none)* | Requires `RenderFormat` ✓ |
| `RRect` | *(none)* | Requires `RenderFormat` ✓ |
| `Oval` | *(none)* | Requires `RenderFormat` ✓ |
| `TabControl` | `TabControlObj` | |
| `TabPanel` | *(none)* | Header only — child of `TabControlObj` |
| `SlideControl` | `SlideControlObj` | |
| `SlidePanel` | *(none)* | Header only — child of `SlideControlObj` |
| `PopoverButton` | `PopoverButtonObj` | |
| `Popover` | `PopoverObj` | Child of `PopoverButtonObj` |
| `ExternalObject` | `ExternalObj` | WebViewer (`WEBV`); Chart (`CHRT`) not generatable |
| `Graphic` | `GraphicObj` | Image data not portable via clipboard |

---

## §4 Styles

Minimal form — sufficient for generation: ✓

```xml
<Styles>
  <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
</Styles>
```

`FullCSS` not required — FM computes it from `ThemeName` + `LocalCSS` at paste time. ✓

These are **round-trip artifacts** — FM adds them on export but does not require them on paste. Omit when generating:
- `FullCSS` — FM computes from `ThemeName` + `LocalCSS` ✓
- `ExtendedAttributes` on `ExternalObj` (WebViewer) — not required and not added on paste; omit ✓
- `ExtendedAttributes` on `FieldObj` — FM generates from field type and formatting settings. Also inert for §31 purposes (including on `PlaceholderText` fields — the protection is sink-side, on `TextObj`s), so omission is safe in every case ✓
- `DDRInfo` — FM populates from the file's own field registry ✓
- `ParagraphStyleVector` — FM adds on export; not required for paste ✓
- `SlidePanel > Styles` — FM 26 exports SlidePanels with no `Styles` element at all; not required for paste ✓
- Empty `<HideCondition>` elements (ButtonBar segments `findMode="True"`, some portal fields `findMode="False"`) — FM adds on export; omit ✓

**Exception: `ExtendedAttributes` on a `TextObj` is required, not optional, whenever more than one `TextObj`-bearing object is being pasted in the same operation.** Omitting it is the root cause of the multi-object text corruption in §31 — include it on every generated Text/Button object. This is the one item in this list that is not safe to omit.

With style override:

```xml
<Styles>
  <LocalCSS>
self:normal .self
{
    background-color: rgba(20%,20%,20%,1);
    color: rgba(100%,100%,100%,1);
}
  </LocalCSS>
  <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
</Styles>
```

`LocalCSS` before `ThemeName`. Include only properties that differ from the theme default.

---

## §5 Field

```xml
<Object type="Field" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="10" left="10" bottom="30" right="200"/>
  <FieldObj numOfReps="1" flags="0" inputMode="0" keyboardType="1"
            displayType="0" quickFind="1" pictFormat="5">
    <Name>TableOccurrence::FieldName</Name>
    <Styles>
      <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
    </Styles>
  </FieldObj>
</Object>
```

### §5.1 FieldObj attributes

| Attribute | Default | Notes |
|---|---|---|
| `numOfReps` | `1` | Repetitions to display. An unresolvable field name resets this to 1 on paste ✓ |
| `flags` | `0` | See §5.2 |
| `inputMode` | `0` | Input method |
| `keyboardType` | `1` | Touch keyboard type |
| `displayType` | `0` | Control style — see §5.3 |
| `quickFind` | `1` | `0` = excluded. Mirrors flags bit 15 |
| `pictFormat` | `5` | Container display format |

### §5.2 FieldObj flags

| Bit | Value | Meaning |
|---|---|---|
| 0 | 1 | Include other value (radio/checkbox sets) — adds an "Other..." entry with its own checkbox/radio button, confirmed on both display types ✓ |
| 1 | 2 | **Select entire contents of field on entry.** Confirmed additive across all four field-entry access states (Edit, Select Only, View Only, Set by Calculation) — the Field Behavior "Select contents on entry" checkbox. Independent of every other bit; applies even to View Only fields (contents can still be select-and-copied on entry even when not editable) ✓ |
| 2 | 4 | Not enterable in Browse mode — also the low bit of the Browse-mode access-state pair, see decode below ✓ |
| 4 | 16 | Not enterable in Find mode — the Find-mode counterpart of bit 2, see decode below ✓ |
| 5 | 32 | Tab to next object ✓ |
| 10 | 1024 | Calendar popup button (with bit 19) ✓ |
| 11 | 2048 | Auto-complete using existing values ✓ |
| 15 | 32768 | Quick Find off — also sets `quickFind="0"` ✓ |
| 19 | 524288 | Calendar popup button (with bit 10) ✓ |
| 20 | 1048576 | Edit box marker — set when displayType=0 ✓ |
| 24 | 16777216 | Field access-state marker — Browse mode, high bit of the Select Only pair, see decode below ✓ |
| 25 | 33554432 | Field access-state marker — Find mode, high bit of the Select Only pair, see decode below ✓ |

### §5.2.1 Field entry access states — full decode

Browse mode and Find mode each have an independent access-state setting (Edit / View Only / Select Only / Set by Calculation), encoded as a bit-pair one apart — 2/4 for the View Only state, 24/25 for the Select Only state:

| Browse mode | Find mode | FieldObj flags bits set (beyond baseline 5,20) |
|---|---|---|
| Edit | Edit | none (baseline) |
| View Only | Edit | bit 2 |
| Select Only | Edit | bit 24 |
| Set by Calculation | Edit | bit 2 + bit 24, plus `CanEntryCalc` |
| Edit | View Only | bit 4 |
| Edit | Select Only | bit 25 |
| Edit | Set by Calculation | bit 4 + bit 25, plus `CanEntryCalc` |

Both dropdowns follow the identical pattern in their own mode: View Only sets the low bit alone, Select Only sets the high bit alone, Set by Calculation sets both bits together. **A single `CanEntryCalc` element governs whichever mode(s) are set to Set by Calculation** — there is only one calculation on the field, shared between modes, not a separate element per mode. See §27 for the `CanEntryCalc` element itself. ✓

Common combinations:
- `0` — default
- `32` — tab only
- `36` — not enterable + tab ✓ (survives verbatim standalone and in portals; behaviour confirmed by same-paste portal comparison against a `32` field — §10.5)
- `32804` — not enterable + tab + Quick Find off ✓
- `32800` — tab + Quick Find off ✓
- `525344` — tab + calendar button (bits 5,10,19) ✓
- `1048608` — tab + edit box marker (bits 5,20) ✓
- `1048610` — tab + edit box marker + select on entry (bits 1,5,20) ✓
- `1048612` — tab + edit box marker + View Only, Browse (bits 2,5,20) ✓
- `17825824` — tab + edit box marker + Select Only, Browse (bits 5,20,24) ✓
- `17825828` — tab + edit box marker + Set by Calculation, Browse (bits 2,5,20,24) ✓
- `1048624` — tab + edit box marker + View Only, Find (bits 4,5,20) ✓
- `34603040` — tab + edit box marker + Select Only, Find (bits 5,20,24,25 — Find-mode Select Only stacks on top of Browse-mode Select Only in this capture) ✓
- `34603056` — tab + edit box marker + Set by Calculation, Find, over Select Only Browse (bits 2,4,5,20,24,25) ✓

### §5.3 FieldObj displayType

| Value | Control |
|---|---|
| `0` | Edit box ✓ |
| `1` | Drop-down list ✓ |
| `2` | Pop-up menu ✓ |
| `3` | Checkbox set ✓ |
| `4` | Radio button set ✓ |
| `5` | Not a functional control ✓ — pastes without error but the field's `Styles` element comes back completely empty (no `FullCSS` generated at all), unlike every valid `displayType`. FM tolerates the value but does not render a real control for it. Do not generate. |
| `6` | Drop-down Calendar ✓ |

`displayType=6` applies to any field type that supports the control (text, number, date, time, timestamp). Container, calculation, and summary fields do not support it and remain at `displayType=0`. The calendar popup icon within the control is a separate option — see `FieldObj flags` bits 10+19. ✓

**Value list binding.** The control type (`displayType`) is independent of the value-list binding; a `displayType=1` field may have no value list at all. To bind a value list, emit it in **two** places: a `<ValueList>NAME</ValueList>` child of `FieldObj` (immediately after `Name`), AND a `<ValueList name="NAME" id="N"/>` descriptor in `DDRInfo`. Both are required — a `DDRInfo` descriptor alone does not attach on paste. The `id` is a small integer (theme/file-assigned), not a UUID. ✓

```xml
<FieldObj displayType="1" ...>
  <Name>TO::FieldName</Name>
  <ValueList>MyValueList</ValueList>
  <Styles>...</Styles>
  <DDRInfo>
    <Field name="FieldName" id="6" repetition="1" maxRepetition="1" table="TO"/>
    <ValueList name="MyValueList" id="2"/>
  </DDRInfo>
</FieldObj>
```

A generated field with both forms binds on paste, verified for `displayType` 1 and 2. The `id` must be the value list's real internal id, sourced from a field that already uses it (the `DDRInfo` descriptor of any field bound to that list) — it cannot be derived from the name. ✓

### §5.4 `pictFormat` / `graphicFormat`

Both attributes always carry the same value. ✓

| Value | Inspector option |
|---|---|
| `4` | Crop to frame |
| `5` | Reduce image to fit *(default)* |
| `6` | Enlarge image to fit |
| `7` | Reduce or enlarge image to fit |

### §5.5 Name

Use the Table Occurrence name from the Relationships graph. ✓

Name binding is case-insensitive: a generated `untitled::Field` binds to `Untitled::Field` and FM rewrites the canonical case on round-trip. ✓

### §5.6 PlaceholderText (ghost text)

First confirmed via round-trip. Child of `FieldObj`, positioned after `Styles` and before `DDRInfo`: ✓

```xml
<FieldObj numOfReps="1" flags="0" inputMode="0" keyboardType="1"
          displayType="0" quickFind="1" pictFormat="5">
  <Name>TableOccurrence::FieldName</Name>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
  <PlaceholderText findMode="True">
    <Calculation><![CDATA["Enter a value here"]]></Calculation>
  </PlaceholderText>
</FieldObj>
```

Generated with no `findMode` attribute; FM added `findMode="True"` itself on round-trip. Sets `FieldObj` flags bit 17 (131072). Confirmed working end to end: the ghost text displays in the empty field exactly as specified. ✓

**§31 accumulation: source confirmed, sink-side fix confirmed.** A field with `PlaceholderText` followed by a `Text` object lacking `ExtendedAttributes` produced a Text object contaminated with the field's literal `Calculation` source (quotes included) prefixed onto its own content — `PlaceholderText`'s calculation feeds the same running accumulator as `TextObj`. ✓

The standing §31 fix covers it entirely. With `ExtendedAttributes` on every `TextObj` in the paste, multiple `PlaceholderText` fields combine freely with each other and with Text/Button objects: verified at two placeholder fields + Text (full round-trip, both Calculations verbatim, canary clean) and at four placeholder fields interleaved with two Texts and a Button (all four ghosts rendering, both canaries and button label clean). ✓

`ExtendedAttributes` on the `FieldObj` itself is **inert** for this purpose — a controlled run with bare FieldObjs and an EA-carrying downstream Text was equally clean, placeholders intact. The protection is entirely sink-side, and unlike a non-front `TabPanel`'s dynamic `TitleCalc` (§11), the placeholder source is not consumed when its discharge path is blocked. ✓

**Review flag (unchanged FM behaviour):** a `PlaceholderText`-bearing field alongside any EA-less `TextObj` in the same snippet is still the original corruption. The generation rule prevents it; flag it when reviewing third-party XML.

### §5.7 Portal field bounds

Fields inside a portal use **relative** bounds. First data row starts at `top="4"`. ✓  
Header-row fields use `top="-1"` to sit above the scrolling area — a generated header field pastes correctly positioned. ✓

---

## §6 Text

```xml
<Object type="Text" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="10" left="10" bottom="25" right="200"/>
  <TextObj flags="0">
    <ExtendedAttributes fontHeight="10" graphicFormat="0">
      <NumFormat flags="0" charStyle="0" negativeStyle="0" currencySymbol="" thousandsSep="0" decimalPoint="0" negativeColor="#0" decimalDigits="0" trueString="" falseString=""/>
      <DateFormat format="0" charStyle="0" monthStyle="0" dayStyle="0" separator="0">
        <DateElement>0</DateElement>
        <DateElement>0</DateElement>
        <DateElement>0</DateElement>
        <DateElement>0</DateElement>
        <DateElementSep index="0"/>
        <DateElementSep index="1"/>
        <DateElementSep index="2"/>
        <DateElementSep index="3"/>
        <DateElementSep index="4"/>
      </DateFormat>
      <TimeFormat flags="0" charStyle="0" hourStyle="0" minsecStyle="0" separator="0" amString="" pmString="" ampmString=""/>
      <CharacterStyle mask="32695">
        <Font-family codeSet="Roman" fontId="0" postScript="Helvetica">Helvetica</Font-family>
        <Font-size>12</Font-size>
        <Face>0</Face>
        <Color>#000000</Color>
      </CharacterStyle>
    </ExtendedAttributes>
    <Styles>
      <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
    </Styles>
    <CharacterStyleVector>
      <Style>
        <Data>Label text</Data>
        <CharacterStyle mask="32695">
          <Font-family codeSet="Roman" fontId="0" postScript="Helvetica">Helvetica</Font-family>
          <Font-size>12</Font-size>
          <Face>0</Face>
          <Color>#000000</Color>
        </CharacterStyle>
      </Style>
    </CharacterStyleVector>
    <ParagraphStyleVector>
      <ParagraphStyle>
        <Alignment type="Left"/>
        <LeftMargin>0</LeftMargin>
        <RightMargin>0</RightMargin>
        <FirstLineIndent>0</FirstLineIndent>
      </ParagraphStyle>
    </ParagraphStyleVector>
  </TextObj>
</Object>
```

`ExtendedAttributes` is included here deliberately — see §31. Its absence is the root cause of multi-object paste corruption when two or more Text objects are pasted together; always generate it, matching the `CharacterStyle` values to the object's own `CharacterStyleVector`.

### §6.1 TextObj flags

| Value | Meaning |
|---|---|
| `0` | Static label |
| `10` | Merge field — set on both `Object` and `TextObj` ✓ |
| `170` | Merge calculation — not portable via paste |

`Object flags` for merge text is set by FM on paste (comes back as `8`, not `10`). Set both `Object flags="10"` and `TextObj flags="10"` when generating — FM corrects the Object flags itself. ✓

Merge field syntax in `Data` element — use CDATA with literal `<<` and `>>`:
```xml
<Data><![CDATA[Hello <<TO::FieldName>> world]]></Data>
```
Do NOT use XML entities (`&lt;&lt;`). ✓

### §6.2 Face values

| Value | Style |
|---|---|
| `0` | Normal |
| `1` | Bold |
| `2` | Italic |
| `3` | Bold + italic |

---

## §7 Shapes (Line, Rect, RRect, Oval)

```xml
<Object type="Rect" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="10" left="10" bottom="50" right="200"/>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
  <RenderFormat renderType="0"/>
</Object>
```

No inner typed element. Element order: `Bounds` → `Styles` → `RenderFormat`. ✓

Use `flags="0"` for all shapes. FM determines line direction from `Bounds` coordinates — bits 12 and 13 appear on both horizontal and vertical lines; neither is a direction indicator. ✓

Empty `SortList` element required even when no sort configured. ✓

---

## §8 Button

```xml
<Object type="Button" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="10" left="10" bottom="35" right="120"/>
  <TextObj flags="2">
    <ExtendedAttributes fontHeight="10" graphicFormat="0">
      <NumFormat flags="0" charStyle="0" negativeStyle="0" currencySymbol="" thousandsSep="0" decimalPoint="0" negativeColor="#0" decimalDigits="0" trueString="" falseString=""/>
      <DateFormat format="0" charStyle="0" monthStyle="0" dayStyle="0" separator="0">
        <DateElement>0</DateElement>
        <DateElement>0</DateElement>
        <DateElement>0</DateElement>
        <DateElement>0</DateElement>
        <DateElementSep index="0"/>
        <DateElementSep index="1"/>
        <DateElementSep index="2"/>
        <DateElementSep index="3"/>
        <DateElementSep index="4"/>
      </DateFormat>
      <TimeFormat flags="0" charStyle="0" hourStyle="0" minsecStyle="0" separator="0" amString="" pmString="" ampmString=""/>
      <CharacterStyle mask="32695">
        <Font-family codeSet="Other" fontId="0" postScript="HelveticaNeue">Helvetica Neue</Font-family>
        <Font-size>16</Font-size>
        <Face>0</Face>
        <Color>#0091CE</Color>
      </CharacterStyle>
    </ExtendedAttributes>
    <Styles>
      <!-- Any LocalCSS for this button's appearance (.self, .icon, .inner_border, etc.) goes here, inside TextObj > Styles -->
      <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
    </Styles>
    <CharacterStyleVector>
      <Style>
        <Data>Button Label</Data>
        <CharacterStyle mask="32695">
          <Font-family codeSet="Other" fontId="0" postScript="HelveticaNeue">Helvetica Neue</Font-family>
          <Font-size>16</Font-size>
          <Face>0</Face>
          <Color>#0091CE</Color>
        </CharacterStyle>
      </Style>
    </CharacterStyleVector>
  </TextObj>
  <ButtonObj buttonFlags="0" iconSize="0" displayType="0">
    <Step enable="True" id="1" name="Perform Script">
      <Script id="1" name="ScriptName"/>
    </Step>
  </ButtonObj>
</Object>
```

Button label text lives in `TextObj > CharacterStyleVector > Style > Data`. `TextObj` is required on buttons. `LabelCalc` isn't functionally used for a static label — FM adds an empty one as a round-trip artifact (§19.1). **Render verified on FM 26:** a single generated Button with its label in `Data` pastes, renders the label correctly, and round-trips verbatim (FM adds the matching `ParagraphStyleVector` and strips a generated `Step id="0" name="None"` to an empty `ButtonObj`). The Data-label render failure is confined to ButtonBar segments (§9.1); standalone Buttons and PopoverButtons (§14) both genuinely label via `Data`. ✓ `ExtendedAttributes` is included above for the same reason as §6 — its absence caused multi-object Text paste corruption (§31); confirmed necessary for Button objects too, same mechanism (`TextObj`).

**Button styling placement, corrected.** All LocalCSS/FullCSS for a standalone Button's appearance — `.self`, `.icon`, `.inner_border`, and every other selector — must be generated inside `TextObj > Styles`, not in a separate object-level `Styles` block after `ButtonObj`. A `.self` background-color and `.icon` color placed in `TextObj > Styles` both survived, merged into `FullCSS`, and rendered correctly (confirmed with a pink button and a magenta icon in the same paste). The identical properties placed in an object-level `Styles` block after `ButtonObj` were silently dropped in three separate attempts — no error, no trace in the returned XML. There is no object-level `Styles` block on a standalone Button; do not generate one. See §23.

**TextObj flags on standalone Buttons normalise to `2`, not `0`.** FM 26 returns `TextObj flags="2"` regardless of what was generated, across every standalone Button capture. Generate `flags="2"` to match native shape. ✓

### §8.1 ButtonObj attributes

| Attribute | Values |
|---|---|
| `buttonFlags` | Bitmask. Bit 0 (1) = calculated label — set by FM when a non-literal `LabelCalc` expression applies ✓; NOT set for quoted-literal LabelCalcs, which return `buttonFlags="2"` on both native and generated FM 26 segments ✓. Bit 1 (2) = round-trips intact ✓ (present on all observed FM 26 segments; earlier noted as toggle). `3` = both bits. Generate `0` on standalone buttons; on segments the native shape is `buttonFlags="2" iconSize="16"` (§9). |

**Segment icons (FM 26, verified by per-segment capture).** An icon on a segment is carried entirely inside `ButtonObj` — segment `Object flags` are untouched. Components:

- `displayType` on the segment `ButtonObj` selects the label/icon arrangement, fully derived by capturing one bar per Button Bar Setup arrangement option in dialog order: `0` = label only, `1` = icon only, `2` = icon above label, `3` = label above icon, `4` = icon left of label, `5` = icon right of label ✓. The icon streams are arrangement-independent (identical FNAM/GLPH/SVG across all values), and `iconSize` persists dormant at `displayType="0"` ✓
- Three `Stream` children follow inside `ButtonObj`: `FNAM` (hex-encoded icon/font name record), `GLPH` (single byte `01`), and `SVG ` (hex-encoded SVG source with an `fm_fill` class group) — one triple per segment, each segment can carry a different icon ✓
- `iconSize` sets the icon point size ✓

See §19 for the SVG stream format detail.
| `iconSize` | `0`–`19` |
| `displayType` | `0`–`4` (text/icon display mode) |

### §8.2 Layout object name

The object name is the **`name` attribute on the `Object` element** — generate it there and it binds and round-trips on standalone Buttons, ButtonBars, individual bar segments, and WebViewers. ✓ FM does not add flags bit 16 to paste-generated named objects; the attribute alone is the mechanism.

Do not confuse this with `TextObj > Styles > CustomStyles > Name` — that element is named-**style** binding by `FM-`UUID (§25.3), not the object name.

### §8.3 HideCondition

`HideCondition` is the LAST behavioural child of the `Object`, after the entire typed inner element block. On shapes FM emits it **before** `RenderFormat` on round-trip; generating it after `RenderFormat` also pastes correctly — FM normalises. Applies to any object type. When `CanEntryCalc` is also present, the order is `CanEntryCalc` then `HideCondition`. See §21. ✓

```xml
<Object type="Field" key="1" flags="4" rotation="0">
  <Bounds .../>
  <FieldObj ...>
    ...
    <Styles>...</Styles>
    <DDRInfo>...</DDRInfo>
  </FieldObj>
  <HideCondition findMode="False">
    <Calculation><![CDATA[IsEmpty($$var)]]></Calculation>
  </HideCondition>
</Object>
```

`findMode="False"` = hide in Browse only (default).
`findMode="True"` = hide in Find mode. ✓
Sets `Object flags` bit 2 (value 4). ✓

---

## §9 ButtonBar

**FM 26 label mechanism: `LabelCalc` per segment, NOT `CharacterStyleVector > Data`.** Segment label text placed in `Data` survives round-trip byte-perfect but renders blank — see §9.1. The structure below is the verified generation pattern, matching native FM 26 export shape (native capture + generated paste rendering + full round-trip, all confirmed):

```xml
<Object type="ButtonBar" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="0" left="0" bottom="35" right="300"/>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
  <ButtonBarObj flags="0" segmentKey="0">
    <Object type="Button" key="2" LabelKey="0" flags="8" rotation="0">
      <Bounds top="1" left="1" bottom="34" right="150"/>
      <TextObj flags="2">
        <ExtendedAttributes fontHeight="10" graphicFormat="0">
          <!-- standard block, as §31 — REQUIRED -->
        </ExtendedAttributes>
        <Styles>
          <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
        </Styles>
        <CharacterStyleVector>
          <Style>
            <Data/>
            <CharacterStyle mask="32695">
              <Font-family codeSet="Other" fontId="0" postScript="HelveticaNeue">Helvetica Neue</Font-family>
              <Font-size>16</Font-size>
              <Face>0</Face>
              <Color>#0091CE</Color>
            </CharacterStyle>
          </Style>
        </CharacterStyleVector>
      </TextObj>
      <ButtonObj buttonFlags="2" iconSize="16" displayType="0">
</ButtonObj>
      <LabelCalc>
        <Calculation><![CDATA["Home"]]></Calculation>
      </LabelCalc>
    </Object>
    <Object type="Button" key="3" LabelKey="0" flags="8" rotation="0">
      <Bounds top="1" left="150" bottom="34" right="299"/>
      <TextObj flags="2">
        <ExtendedAttributes fontHeight="10" graphicFormat="0">
          <!-- standard block -->
        </ExtendedAttributes>
        <Styles>
          <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
        </Styles>
        <CharacterStyleVector>
          <Style>
            <Data/>
            <CharacterStyle mask="32695">
              <Font-family codeSet="Other" fontId="0" postScript="HelveticaNeue">Helvetica Neue</Font-family>
              <Font-size>16</Font-size>
              <Face>0</Face>
              <Color>#0091CE</Color>
            </CharacterStyle>
          </Style>
        </CharacterStyleVector>
      </TextObj>
      <ButtonObj buttonFlags="2" iconSize="16" displayType="0">
</ButtonObj>
      <LabelCalc>
        <Calculation><![CDATA["Detail"]]></Calculation>
      </LabelCalc>
    </Object>
  </ButtonBarObj>
</Object>
```

- **Segment label in `<LabelCalc>` as the last child of the segment `Object`, after `ButtonObj`** — this is what the FM 26 renderer reads. Generated LabelCalc survives round-trip verbatim and renders. ✓
- `CharacterStyleVector > Data` on segments: leave empty (`<Data/>`), matching native shape. Populated Data survives but does not render — see §9.1. ✓
- Segment Object `flags="8"` (native FM 26 plain segment; see flag table note below) ✓
- `ButtonObj buttonFlags="2" iconSize="16" displayType="0"`, empty — no `Step` child on plain segments; a generated `Step id="0" name="None"` is stripped on paste ✓
- ButtonBar-level `Styles` before `ButtonBarObj` (native export order; FM also accepts it after) ✓
- `ButtonBarObj` requires `flags="0" segmentKey="0"` attributes ✓
- `TextObj flags="2"` inside ButtonBar segments (not `"0"`) ✓
- Button segment bounds start at `(1,1)` not `(0,0)` — FM adds 1pt inset ✓
- Button segments are adjacent: second button's `left` = first button's `right` ✓
- Round-trip artifacts FM may add inside segments (omit when generating): empty `<HideCondition findMode="True">`, and inside a `Step`, `<DisableStepCollapsed state="False"/>` and `<CurrentScript value="Pause"/>` ✓

### §9.1 Survives-but-does-not-render: the Data label trap

Segment label text generated in `CharacterStyleVector > Style > Data` (and mirrored in `ParagraphStyleVector`) pastes without error, survives copy-back **byte-perfect including FM adding matching ParagraphStyleVector entries**, and renders as a blank segment in both Layout and Browse mode. The XML passes any structural audit; only a visual check catches it. Confirmed on FM 26 at three segments generated (blank) against native capture (labels in `LabelCalc`, `Data` empty) and generated `LabelCalc` (renders). ✓

Pinned to FileMaker Pro 26.0.1.51 (macOS). Whether earlier FM releases used the same render path is not established. ○

**Button Object flags in ButtonBar:**

**Verified current (FM 26):** plain segment `flags="8"` — native capture and generated round-trip, renders correctly. ✓ Use `8` when generating.

The table below predates the §9.1 finding and came from the same capture generation as the dead Data mechanism. Treat as legacy observations pending re-derivation from current native captures (icon, active, and named segment variants needed):

| Value | Bits | Use |
|---|---|---|
| `8` | 3 | The segment flag value on FM 26 — the only one. Verified across a plain segment, a segment in a bar with a UI-typed object name, a segment designated active via Specify Active Segment, and a single-segment bar: all export `flags="8"` unchanged ✓ |

**What the legacy flag values actually were.** FM 26 does not encode segment state in `Object flags` at all:

- **Active segment** is a bar attribute: Specify Active Segment (fixed) serialises as `ButtonBarObj segmentKey="<active segment's key>"`; `segmentKey="0"` means none set. Verified by UI round trip — designating segment four returned `segmentKey` pointing at its key while the segment stayed `flags="8"`. ✓ Generate `segmentKey="0"` unless a fixed active segment is wanted.
- **Object names** live purely in the `name` attribute and set no flag bit — verified both for paste-generated names (earlier) and for a UI-typed name on a bar, which exported `name` with `flags="0"`. ✓ The legacy `65544`/`65545` values (bit 16) reflect a mechanism FM 26 does not use.
- **No FM 26 segment state is encoded in `Object flags`.** Four segments with distinct icons all exported `flags="8"`, in both icon-only and icon-with-text arrangements. Never generate `260`, `261`, `256`, `65544` or `65545`. ✓

**Icon colour confirmation.** A native-origin icon capture (Button Setup–added icon, `displayType="4"`, icon left of label) confirms the `-fm-icon-color` property renders correctly when placed in `TextObj > Styles > LocalCSS` under `.icon`, matching the standalone-Button placement rule in §8. Segment Object flags remained `flags="8"` regardless of icon presence. ✓

---

## §10 Portal

```xml
<Object type="Portal" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="100" left="10" bottom="300" right="400"/>
  <PortalObj portalFlags="21" numOfRows="5" initialRow="1">
    <TableAliasKey>TableOccurrenceName</TableAliasKey>
    <SortList>
      <Sort type="Ascending">
        <Name>TableOccurrenceName::FieldName</Name>
      </Sort>
    </SortList>
    <FilterCalc>
      <Calculation><![CDATA[TableOccurrenceName::Status = "Active"]]></Calculation>
    </FilterCalc>
    <Styles>
      <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
    </Styles>
    <Object type="Field" key="2" LabelKey="0" flags="0" rotation="0">
      <Bounds top="4" left="4" bottom="35" right="150"/>
      <FieldObj numOfReps="1" flags="32" inputMode="0" keyboardType="1"
                displayType="0" quickFind="1" pictFormat="5">
        <Name>TableOccurrenceName::FieldName</Name>
        <Styles>
          <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
        </Styles>
      </FieldObj>
    </Object>
  </PortalObj>
</Object>
```

### §10.1 PortalObj attributes

| Attribute | Notes |
|---|---|
| `portalFlags` | See §10.2 |
| `numOfRows` | Rows to display |
| `initialRow` | `1` = from beginning |

### §10.2 portalFlags

**Bit 0 = Allow Vertical Scrolling, bit 2 = Allow Deletion of Portal Records.** Both confirmed by direct isolation: a five-portal batch on one relationship, one bit changed per portal against a same-batch baseline, Portal Setup checked on each. ✓

**Bit 8 = "Show scroll bar: When Scrolling"** (versus the default "Always" when bit 8 is absent) — but this effect is conditional: it only appears when bit 0 (Allow Vertical Scrolling) is also set. Isolating bit 8 alone against an all-off baseline (`flags=272`) showed no effect, because the "Show scroll bar" dropdown is inactive when scrolling itself is off — the bit's effect was real but hidden behind a disabled control. Confirmed by testing `flags=273` (bits 0+8 together): the dropdown reads "When Scrolling" instead of "Always". ✓

**Bits 1, 5, 6, and 9 have no effect.** Verified with scrolling and deletion active (ruling out a dependency on another bit), and with Sort and Filter working and their Specify dialogs inspected directly (ruling out an effect hidden in a sub-dialog). All five portals identical throughout, including field selection, order direction, custom order, reorder-by-summary and language override inside Sort's Specify dialog. ✓

**Sort and filter require both the real content AND the matching bit set together — neither alone is sufficient.** A portal with genuine `SortList`/`FilterCalc` content but bits 3/7 absent from `portalFlags` comes back with "Sort/Filter portal records" unticked in Portal Setup, even though the content itself survives the round-trip intact. But a portal generated with the same real content **and** bits 3/7 explicitly included in `portalFlags` (confirmed at `flags=397`) comes back with both correctly ticked. ✓ Content-only and bit-only are each insufficient; the combination is required and is achievable by generating XML directly — no UI step needed.

```xml
<PortalObj portalFlags="397" numOfRows="4" initialRow="1">
  <TableAliasKey>apple</TableAliasKey>
  <SortList>
    <Sort type="Ascending">
      <Name>apple::field1</Name>
    </Sort>
  </SortList>
  <FilterCalc>
    <Calculation><![CDATA[1=1]]></Calculation>
  </FilterCalc>
  ...
</PortalObj>
```

**"Use alternate row state" and "Use active row state" are controlled via `LocalCSS`, not `portalFlags`.** Confirmed directly: setting `-fm-portal-alt-background: false` unticks "Use alternate row state" in Portal Setup, and `-fm-use-portal-current-row-style: false` unticks "Use active row state" — both properties sit in the same `self:normal .self` block. Both default to `true` (both checkboxes ticked) when the properties are absent, which is why they appeared ticked in every `portalFlags` bit-isolation test regardless of the flags value — none of those bits were ever the relevant control. ✓

```xml
<Styles>
  <LocalCSS>
self:normal .self
{
	-fm-portal-alt-background: false;
	-fm-use-portal-current-row-style: false;
}
  </LocalCSS>
  <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
</Styles>
```

| Bit | Value | Meaning |
|---|---|---|
| 0 | 1 | Allow Vertical Scrolling ✓ |
| 2 | 4 | Allow Deletion of Portal Records ✓ |
| 3 | 8 | Sort — requires real `SortList` content AND this bit set together; neither alone is sufficient ✓ |
| 4 | 16 | Reset Scroll Bar When Exiting Record — **inverted logic**: bit absent → checkbox ticked; bit present → checkbox unticked. Confirmed by direct same-batch comparison (`flags=273` vs `flags=257`). Not a required flag — the portal functions normally with or without it. ✓ |
| 7 | 128 | Filter — requires real `FilterCalc` content AND this bit set together; neither alone is sufficient ✓ |
| 8 | 256 | "Show scroll bar: When Scrolling" instead of "Always" — only has an effect when bit 0 is also set ✓ |
| 1, 5, 6, 9 | 2, 32, 64, 512 | No effect — confirmed with scrolling+deletion active and separately with Sort/Filter genuinely working, including inside Sort's Specify dialog ✓ |

`TableAliasKey` must reference an actual related table occurrence; pointing it at the portal's own base table gives no valid relationship context, and any `Sort`/`FilterCalc` content is discarded regardless of the flags value. ✓

Every isolation test in this section held bit 4 constant (present) across both the baseline and the test portal in each comparison, so the bit 0/2/8 conclusions above are unaffected by this — only the label "bit 4" and the description of `flags=16` as a "bare/base" portal needed correcting.

| Value | Bits | Scenario |
|---|---|---|
| `16` | 4 | Reset Scroll Bar unticked (bit 4 present), nothing else set |
| `17` | 0,4 | Allow vertical scrolling, "Show scroll bar: Always", Reset unticked ✓ |
| `20` | 2,4 | Allow deletion of portal records, Reset unticked ✓ |
| `21` | 0,2,4 | Scrolling + deletion, Reset unticked |
| `257` | 0,8 | Scrolling, "When Scrolling", **Reset Scroll Bar ticked** (bit 4 absent) ✓ |
| `273` | 0,4,8 | Scrolling, "When Scrolling", Reset unticked (bit 4 present) ✓ |
| `397` | 0,2,3,7,8 | Sort + filter (with matching real content) + scrolling ("When Scrolling") + deletion, Reset ticked (bit 4 absent) — every bit-level prediction in this section confirmed simultaneously in one composite portal ✓ |
| `157` | 0,2,3,4,7 | Sort + filter + scrolling + deletion, Reset unticked, as exported from a hand-configured portal ✓ |
| `401` | 0,4,7,8 | Allow Vertical Scrolling + Filter (real content + bit) + Reset Scroll Bar unticked + Show scroll bar: When Scrolling. Confirmed by direct construction matching this exact bit prediction, using a real relationship and a real `FilterCalc` ✓. The "planner/single-row display" label attached to earlier captures does not describe a distinct mechanism — it was a mislabelled ordinary combination of already-documented bits. |



Empty (required even with no sort): `<SortList>
</SortList>` ✓

Sort structure — works when paired with `portalFlags` bit 3 (see §10.2):
```xml
<SortList>
  <Sort type="Ascending">
    <Name>TableOccurrenceName::FieldName</Name>
  </Sort>
</SortList>
```

`type="Custom"` for "Custom order based on value list" — confirmed structure, native export: ✓
```xml
<SortList>
  <Sort type="Custom">
    <Name>TableOccurrenceName::FieldName</Name>
    <ValueList>ValueListName</ValueList>
  </Sort>
</SortList>
```
Unlike field-level value list binding (§5.3), which requires both the `<ValueList>` element and a `DDRInfo` id descriptor, this Sort-level `<ValueList>` carries only the name — no id anywhere. Do not add a `DDRInfo` id here by analogy with §5.3; the two contexts bind differently. ✓

`TableAliasKey` referencing the portal's own base table (no real relationship) drops this content entirely on paste — confirmed: a `SortList` with real content came back completely empty in this scenario. ✓ `FilterCalc` behaves slightly differently in the same scenario — its content survives the round-trip intact rather than being stripped, but remains non-functional (no working filter, consistent with §10.2). Against a valid relationship, `SortList`/`FilterCalc` content alone (with bit 3 absent from `portalFlags`) survives the round-trip cosmetically but "Sort portal records" stays unticked. Generating this element **together with** `portalFlags` bit 3 set produces a working, ticked sort — confirmed via `portalFlags="397"`. ✓ `type="Ascending"`/`type="Descending"` is honored exactly — confirmed via the Sort Records dialog showing the correct field and correct radio button selected for `type="Descending"`. ✓

### §10.4 FilterCalc

Omit entirely when no filter. Works when paired with `portalFlags` bit 7 (see §10.2):
```xml
<FilterCalc>
  <Calculation><![CDATA[TableOccurrenceName::Status = "Active"]]></Calculation>
</FilterCalc>
```
Content alone (bit 7 absent) survives the round-trip but "Filter portal records" stays unticked — same pattern as `SortList`. Generating this element together with `portalFlags` bit 7 set produces a working, ticked filter — confirmed via `portalFlags="397"`. ✓

Element order in `PortalObj`: `TableAliasKey` → `SortList` → `FilterCalc` → `Styles` → field `Object` elements. Generated in this order against a real relationship (flags 397, live sort and filter content), returned in this order verbatim. ✓

### §10.5 Portal field FieldObj flags

| Value | Meaning |
|---|---|
| `32` | Enterable ✓ — generated portal field accepts entry in Browse mode; flags value round-trips verbatim |
| `36` | Not enterable ✓ — generated portal field is view-only in Browse mode; flags value round-trips verbatim. Same-paste comparison against a `32` sibling field |

---

## §11 TabControl / TabPanel

```xml
<Object type="TabControl" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="50" left="10" bottom="400" right="600"/>
  <TabControlObj tabHeight="20" visPanelKey="2" defaultVisPanelKey="2"
                 visPanelIndex="0" defaultVisPanelIndex="0"
                 tabWidthModifier="70" tabFlagSet="312">
    <Styles>
      <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
    </Styles>
    <Object type="TabPanel" key="2" LabelKey="0" flags="0" rotation="0">
      <Bounds top="0" left="0" bottom="350" right="590"/>
      <Styles>
        <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
      </Styles>
      <TitleCalc>
        <Calculation><![CDATA["My Tab"]]></Calculation>
      </TitleCalc>
      <TabPanelObj tabLeftEdge="0" tabWidth="100" tabPanelFlagSet="1"/>
    </Object>
  </TabControlObj>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
</Object>
```

Element order in `TabPanel`: `Bounds` → `Styles` → `TitleCalc` → `TabPanelObj`. ✓
`TabControlObj` requires its own `Styles` block before the panel objects. ✓
`TabPanelObj` carries attributes `tabLeftEdge`, `tabWidth`, `tabPanelFlagSet`. If omitted from a generated panel, FileMaker synthesises it on paste and the tab control renders normally, so it is not mandatory to emit; include it when you need to control those attributes. ✓

**TabPanel content nests INSIDE `TabPanelObj`.** Verified in both directions on FM 26: content attached in the UI serialises as child `Object` elements inside the panel's `TabPanelObj` (with bounds relative to the TabControl origin), and a generated `Object` nested inside `TabPanelObj` pastes as attached panel content — it highlights with the panel and hides when another panel is front. Full round-trip: the generated nested child returns still nested, with its control-relative `Bounds` byte-identical. ✓

```xml
<TabPanelObj tabLeftEdge="100" tabWidth="100" tabPanelFlagSet="0">
  <Object type="Text" key="5" ...>
    <Bounds top="100" left="40" bottom="125" right="240"/>  <!-- relative to TabControl origin -->
    ...
  </Object>
</TabPanelObj>
```

**Geometric overlap attaches nothing.** A sibling object generated at absolute coordinates inside the panel area pastes as a free-floating layout object in front of the control, not as panel content. ✓

### §11.1 TabControlObj attributes

| Attribute | Notes |
|---|---|
| `tabHeight` | Derived — FM recomputes from font and padding on paste (a generated `20` comes back `46`); emit any value ✓ |
| `visPanelKey` | Key of currently visible panel. Confirmed independent of `defaultVisPanelKey`: a control copied with panel 2 front but panel 1 as Default Front Tab exports `visPanelKey`/`visPanelIndex` pointing at panel 2 and `defaultVisPanelKey`/`defaultVisPanelIndex` at panel 1, both intact ✓ |
| `defaultVisPanelKey` | Key of the Default Front Tab — binds from generated XML and round-trips independently of the currently visible panel ✓ |
| `visPanelIndex` / `defaultVisPanelIndex` | 0-based indices, survive verbatim, track their respective keys ✓ |
| `tabWidthModifier` | The stored operand for the Tab Width setting. Persists dormant in modes that do not use it (a control in Label Width mode still carries and displays the last modifier value). Read per side in Margin mode, as a floor in Minimum mode, as the exact width (+2pt, see below) in Fixed mode ✓ |
| `tabJustification` | Does not survive round-trip; do not generate. Values 0/1/2 on 2-panel controls all returned with the attribute absent from `TabControlObj` and all three rendered identically, tabs left-aligned at their own width. ✓ |
| `tabFlagSet` | Encodes the Tab Width mode as an enumeration on base `264`, stepping by 16 in dialog order — see table below ✓ |

### §11.1.1 tabFlagSet — Tab Width mode enumeration

All five values round-trip verbatim, display the matching mode in Tab Control Setup, and render the predicted widths and behaviour. Verified with modifier 150 (and Minimum re-verified at modifier 70; Fixed re-verified with a fully typed, drag-free generation at modifier 100):

| `tabFlagSet` | Tab Width mode | Uses modifier | Behaviour |
|---|---|---|---|
| `264` | Label Width | no | Each tab sized to its own label — natural widths |
| `280` | Label Width + Margin of | yes, per side | Label width plus 2× modifier |
| `296` | Width of Widest Label | no | All tabs sized to the widest label |
| `312` | Minimum of | yes, as floor | `max(label width, modifier)` per tab |
| `328` | Fixed Width of | yes, exact | All tabs = modifier + 2pt (see below); long labels clip |

`tabLeftEdge` of each tab equals the previous tab's right edge minus 1pt (e.g. widths 150/150 give edges 0, 149) — adjacent tabs overlap by 1pt, the same inset convention as ButtonBar segments ✓

**Fixed Width of returns `tabWidth = modifier + 2`, confirmed as a real constant, not measurement error.** Verified twice independently: once via manual drag-to-size (150 requested → 152 returned) and once via a fully typed, drag-free generation with no manual sizing step at all (100 requested → 102 returned, all three panels identical). The +2pt is a structural border/inset allowance FM applies uniformly in Fixed mode. ✓

**Generation default: `tabFlagSet="264"`** (Label Width) with any `tabWidthModifier` value, for natural tabs. `312` silently produces minimum-width behaviour whenever a label is narrower than the modifier, and the difference is invisible when every label exceeds it. Emit `312` or `328` plus a meaningful modifier only when uniform tab widths are specifically wanted. ✓

`328` is Fixed Width of.

### Panel title serialisation

**FM 26 exports every dialog-defined panel title as `TitleCalc`** — front and non-front panels alike, whether the name was typed plainly into Tab Name or entered via Specify. No panel `TextObj`/`Data` form appears in any capture. ✓

**Never generate the `TextObj > CharacterStyleVector > Style > Data` form for a panel title.** It appears in some archived snippets and is decode-only; FM 26 does not produce it.

**Dynamic titles: front panel only.** A non-literal `TitleCalc` on a NON-front panel does not survive generated paste — with `ExtendedAttributes` present throughout the batch it is silently dropped (blank tab); without, its calc source migrates into the next EA-less `TextObj` in the paste, destroying that object's label (§31). Generate dynamic titles only as quoted literals on non-front panels, or accept the loss. ✓

TitleCalc accepts a bare FM expression or a quoted string literal:
```xml
<!-- Static -->
<TitleCalc><Calculation><![CDATA["My Tab"]]></Calculation></TitleCalc>
<!-- Dynamic -->
<TitleCalc><Calculation><![CDATA[Let(n = TO::count; "Tab" & Case(n > 0; " (" & n & ")"))]]></Calculation></TitleCalc>
```

### §11.1.2 Checked-state border overrides (worked example)

Overriding `border-*-style: none` on `self:checked .self` and `self:checked .inner_border` (and the `checkedfocus` equivalents) for a TabPanel survives as a genuine LocalCSS delta and renders — the active tab's border is removable per-state. This confirms `checked`/`checkedfocus` in §25.4's state vocabulary are independently stylable in practice, not just theoretically distinct. The override was retained (not pruned) on paste because it genuinely differs from the theme default (solid); see §25.3 for the delta-pruning behaviour that would otherwise strip a redundant override. ✓

---

## §12 SlideControl / SlidePanel

```xml
<Object type="SlideControl" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="20" left="20" bottom="240" right="400"/>
  <SlideControlObj visPanelKey="3" visPanelIndex="1" dotSize="9" slideFlagSet="0">
    <Styles>
      <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
    </Styles>
    <Object type="SlidePanel" key="2" LabelKey="0" flags="0" rotation="0">
      <Bounds top="0" left="0" bottom="220" right="380"/>
      <SlidePanelObj slidePanelFlagSet="0"/>
    </Object>
    <Object type="SlidePanel" key="3" LabelKey="0" flags="0" rotation="0">
      <Bounds top="0" left="0" bottom="220" right="380"/>
      <SlidePanelObj slidePanelFlagSet="0">
        <Object type="Text" key="5" ...>
          <Bounds top="60" left="50" bottom="85" right="250"/>  <!-- relative to SlideControl origin -->
          ...
        </Object>
      </SlidePanelObj>
    </Object>
  </SlideControlObj>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
</Object>
```

SlidePanel `Bounds` are relative to SlideControl. ✓

**SlidePanel content nests INSIDE `SlidePanelObj`** — same mechanism as TabControl (§11), verified in both directions on FM 26: UI-attached content serialises as child `Object` elements inside `SlidePanelObj` with control-relative bounds, and a generated nested `Object` pastes attached and round-trips still nested with `Bounds` byte-identical. ✓ **Geometric overlap attaches nothing** — a sibling object generated at absolute coordinates over the panel area pastes free-floating, exactly as at §11. ✓

**`slideFlagSet` bit 0 (value 1) = navigation dots HIDDEN.** A control generated with `slideFlagSet="1"` pastes with no dots; a control with dots visible exports `slideFlagSet="0"`. Generate `0` for the standard dotted control. ✓

`visPanelKey`/`visPanelIndex` bind from generated XML — a control generated pointing at its second panel returns with that panel's reassigned key and `visPanelIndex` intact. ✓

Native `SlideControlObj` carries its own `Styles` block before the panels (as `TabControlObj` does); a generated control without it still pastes, so it is not mandatory — include a minimal `ThemeName` block to match native shape. **Confirmed:** two controls, one with a full `ThemeName` Styles block inside `SlideControlObj` and one without, produced structurally identical exports (FM synthesised the block onto the second) and both rendered identically with dots visible. ✓ FM 26 exports SlidePanels with NO `Styles` element at all; do not generate one on panels. ✓

No `TitleCalc` — navigation is via dot indicators. A `TitleCalc` generated on a `SlidePanel` is dropped on paste. ✓

---

## §13 GroupButton

```xml
<Object type="GroupButton" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="10" left="10" bottom="35" right="150"/>
  <GroupButtonObj numOfObjs="1">
    <Step enable="True" id="1" name="Perform Script">
      <Script id="1" name="ScriptName"/>
    </Step>
    <Styles>
      <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
    </Styles>
    <Object type="Text" key="2" LabelKey="0" flags="0" rotation="0">
      <Bounds top="10" left="10" bottom="35" right="150"/>
      <TextObj flags="0">...</TextObj>
    </Object>
  </GroupButtonObj>
</Object>
```

`numOfObjs` = count of grouped child objects. Child objects follow `Styles` inside `GroupButtonObj`. Verified at `numOfObjs="3"` with two `Line` children and one `Text` child: count returned verbatim, all three children grouped, child bounds byte-identical. ✓ (Whether FM validates or corrects a deliberately wrong count is untested — keep the count accurate.)

**Position caution.** Generated GroupButton outer `Bounds` are not preserved — FM relocates the outer object on paste (observed twice, including with Paste in Place) while child objects keep their generated absolute coordinates verbatim. Structure (`numOfObjs`, Line children, Text child) survives; placement does not. When position matters, paste the children and group them in the UI. `GroupButtonObj > Styles` returns emptied on round-trip. ✓

GroupButtons can contain `Line` children to draw vector icons:
```xml
<GroupButtonObj numOfObjs="5">
  <Step .../>
  <Styles>...</Styles>
  <Object type="Line" key="2" flags="12288" rotation="0">
    <Bounds top="0" left="0" bottom="0" right="15"/>
    <RenderFormat renderType="0"/>
  </Object>
  <!-- more lines -->
</GroupButtonObj>
```

"Go to Object" step targeting a child field:
```xml
<GroupButtonObj numOfObjs="1">
  <Step enable="True" id="91" name="Go to Object">
    <ObjectName></ObjectName>
    <Repetition></Repetition>
  </Step>
  <Styles>...</Styles>
  <Object type="Field" .../>
</GroupButtonObj>
```

---

## §14 Popover

```xml
<Object type="PopoverButton" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="10" left="10" bottom="35" right="120"/>
  <TextObj flags="2">
    <ExtendedAttributes fontHeight="10" graphicFormat="0">
      <!-- standard block, as §31 — mirror a CharacterStyle; REQUIRED -->
    </ExtendedAttributes>
    <Styles>
      <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
    </Styles>
    <CharacterStyleVector>
      <Style>
        <Data>Open</Data>
        <CharacterStyle mask="32695">...</CharacterStyle>
      </Style>
    </CharacterStyleVector>
  </TextObj>
  <PopoverButtonObj>
    <Object type="Popover" key="2" LabelKey="0" flags="0" rotation="0">
      <Bounds top="50" left="10" bottom="200" right="300"/>
      <Styles>
        <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
      </Styles>
      <TitleCalc>
        <Calculation><![CDATA["My Popover"]]></Calculation>
      </TitleCalc>
      <PopoverObj/>
    </Object>
  </PopoverButtonObj>
  <HideCondition findMode="False">
    <Calculation><![CDATA[IsEmpty($$var)]]></Calculation>
  </HideCondition>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
</Object>
```

Popover `Bounds` are absolute layout coordinates, but FM **recomputes them relative to the button** on paste — generated values are treated as approximate. ✓ `PopoverButtonObj` gains `buttonFlags`/`key`/`iconSize`/`displayType` attributes on round-trip (artifacts; omit when generating).

**The PopoverButton `TextObj` must carry `ExtendedAttributes`** — a bare `<TextObj flags="0"/>` is the §31 leak absorber: in batch testing it received a migrated TitleCalc's raw source as its label. ✓

**PopoverButton label mechanism: `CharacterStyleVector > Data`, NOT `LabelCalc` — the exact inverse of ButtonBar segments (§9).** Fully verified: native FM 26 capture shows label text in `Data` in both vectors with no `LabelCalc` element anywhere; a generated single-object paste with the label in `Data` renders correctly; and the round-trip returns the `Data` verbatim in both vectors with `TitleCalc` intact. ✓ A generated `LabelCalc` is **dropped by the paste handler entirely** — confirmed empty label in both batch pastes and a single-object paste, and a later native export of that pasted button showed no `LabelCalc` element at all. Do not generate `LabelCalc` on PopoverButtons; put the label in `Data`, exactly as a Text object.

Native shape notes: PopoverButton Object `flags="8"` natively (same value as native ButtonBar segments — bit 3 appears on labelled button-family objects); generated `flags="0"` binds and round-trips as `0` ✓. `PopoverObj` carries `flags` / `position` / `key` attributes on export (artifacts; a bare `<PopoverObj/>` pastes ✓).

Element order in `Popover`: `Bounds` → `Styles` → `TitleCalc` → `PopoverObj`. ✓

---

## §15 WebViewer (ExternalObject)

```xml
<Object type="ExternalObject" key="1" LabelKey="0" name="wv1"
        flags="-1073741824" rotation="0">
  <Bounds top="10" left="10" bottom="200" right="500"/>
  <ExternalObj typeID="WEBV" typeIndex="0" externalFlagSet="32865">
    <Styles>
      <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
    </Styles>
    <Calculation index="0"><![CDATA["https://example.com"]]></Calculation>
  </ExternalObj>
</Object>
```

- `flags="-1073741824"` anchors the object on all four sides so it fills the window at any size. Required for a web viewer hosting a single-page app: generating `flags="0"` here produces a viewer pinned to left and top that renders correctly at the design size and then does not grow. For a fixed-size viewer, use `flags="0"`. See §2.2 ✓
- **`name` on the Object element is required for any bridged viewer**, not optional. `Perform JavaScript in Web Viewer` targets by name and `GetLayoutObjectAttribute` reads by name; an unnamed viewer is unreachable in both directions ✓
- `typeIndex` is `0` on every captured WEBV. Do not vary it ◎
- **`ExtendedAttributes` is not required on `ExternalObj` and is not added on paste.** Generated without it, the object pastes clean and returns with no `ExtendedAttributes` element. Omit it. This is a WEBV exception to the §31 rule, which concerns `TextObj` only ✓
- Web viewers carry a 1pt theme border on all four sides. For a full-bleed app host, suppress it per §15.2 ✓
- URL or full HTML string in `Calculation index="0"` — both verified end to end: a quoted URL loads the page, a quoted `data:text/html,...` string renders the HTML including inline `<script>` blocks, and both calculations round-trip verbatim ✓. Size limits are a property of the snippet, not the calculation: see §15.4
- **The Web Viewer Setup preset is not carried by the clipboard format.** Every generated web viewer opens as *Custom Web Address*, whatever the calculation contains. A native Google Maps viewer and a generated object with a byte-identical `Calculation` open on different dialog entries, so the picker does not read the calculation and the selection is stored outside the XML. It is not `typeIndex`, which is `0` in both ✓
- The `/*Address=*/`-style comment markers that FileMaker writes into preset calculations survive verbatim but do not reconstruct the dialog's parameter slots. No reason to generate them ✓
- Chart (`typeID="CHRT"`) not generatable

### §15.1 Web viewer option bits (`externalFlagSet`)

`externalFlagSet` encodes the setup-dialog options, not an opaque constant. Decoded by single-option round-trip testing (FileMaker Pro 26, macOS).

Bit 15 (32768) is present on every WEBV object authored in FileMaker 26. It is **not universal** across older files; see the legacy note below. The option bits are:

| Add | Option / state |
|---|---|
| +1 | Allow interaction with web viewer content — ON |
| +2 | Display content in Find mode — ON |
| +4 | Display progress bar — ON |
| +8 | Display status messages — ON |
| +32 | Automatically encode URL — **OFF** (inverted: ON adds nothing) |
| +64 | Allow JavaScript to perform FileMaker scripts — ON |

Verified in the generation direction: an object generated with `32781` opened in Web Viewer Setup showing exactly the predicted six states. ✓

Confirmed whole values: dialog defaults = `32781`; integrated HTML UI (interaction + JS bridge ON, encode OFF, rest off) = **`32865`**; everything ON = `32847`; JS bridge only = `32864`. ✓

**Analysis rule: read the low bits independently, treat bit 15 as a separate marker.** Objects authored before FM 26 omit it: `externalFlagSet="13"` (bits 0, 2, 3) is the same option set as the FM 26 default `32781` without it. Never decode as `value - 32768`; on a legacy object that goes negative and misreports every option. ✓

**Progress bar and status messages consume the object's height.** `+4` and `+8` draw a progress bar across the top and a status bar along the bottom, and the chrome is a fixed pixel height regardless of object size. On a short web viewer it swallows the content area entirely: at 46pt tall, thirteen viewers at `32781` rendered no page content at all while an identical viewer at `32865` rendered normally. The page loads in every case; there is nowhere to draw it. Only network-loading URLs show the chrome, so a `data:` URL renders regardless. ✓

Generation rule: omit `+4` and `+8` on any small web viewer. The exact height at which the chrome stops mattering was not established; 46pt is confirmed unusable. For a viewer hosting an app UI, omit them at any size.

Generation notes:
- `32865` is the correct configuration for a web viewer hosting an integrated HTML UI, and carries neither `+4` nor `+8`.
- **`+64` gates delivery, not the object's existence.** The `FileMaker` object is present and callable in the page whether or not the bit is set. Without `+64`, `FileMaker.PerformScript()` returns normally, throws nothing, and the call never reaches FileMaker. With it, the call arrives (verified: a call to a non-existent script raised FileMaker's own "This script cannot be found" dialog, while the identical page at `32801` produced nothing at all). ✓

  Debugging consequence: testing `typeof FileMaker` proves nothing. It is defined in both cases. The only reliable check is whether FileMaker actually receives the call.
- For plain URL-display web viewers, generate the dialog default `32781` — shipping the bridge enabled on a viewer rendering arbitrary web content is an unnecessary surface.
- "Automatically encode URL" is inverted (+32 = off). HTML delivered via a data: URL calculation should keep encoding OFF (+32); plain web URLs normally keep it ON (no bit).

---

### §15.2 Border suppression

Web viewers inherit a 1pt solid border on all four sides from the theme. Removing it is a `LocalCSS` override on `border-*-style` only.

- **Set style, never width or colour.** FileMaker retains `border-*-width: 1pt` and the theme border colour in `FullCSS` regardless; both are inert once style is `none`. Emitting width or colour to remove a border is wrong shape and unnecessary. ✓
- **Emit all four sides whenever any side differs from the theme**, including sides that keep `solid`. A single-side override still writes the complete four-side group. ✓
- **Emit `self:normal .self` only.** FileMaker's own export adds a `self:normal .inner_border` block on the all-off case, but a generated one is pruned as redundant against the theme (§25.3 delta-pruning) and the result is identical without it. ✓
- **No focus-state override is required.** The computed `self:focus .self` block carries border colour only, never style, so with normal style at `none` there is nothing for it to colour. ○ (structural; visual confirmation outstanding)

Full-bleed app host. `FullCSS` is omitted because FileMaker recomputes it on paste (§4, §25.2). Round-trip verified from generation ✓:

```xml
<Styles>
  <LocalCSS>
self:normal .self
{
	border-top-style: none;
	border-right-style: none;
	border-bottom-style: none;
	border-left-style: none;
}
</LocalCSS>
  <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
</Styles>
```

A named theme style is the cleaner route where one exists: it can zero `border-*-width` and clear `border-*-color` as well as style, and carry its own `focus` and `hover` handling. See §25.3.

Single edge only, here a left rule. All four sides listed, the wanted edge `solid`, no `.inner_border` block:

```xml
<LocalCSS>
self:normal .self
{
	border-top-style: none;
	border-right-style: none;
	border-bottom-style: none;
	border-left-style: solid;
}
</LocalCSS>
```

---

### §15.3 Behavioural elements and containers on WEBV

FileMaker Pro 26, macOS. Confidence differs by row; read the marks.

| Element | On `ExternalObject` |
|---|---|
| `ConditionalFormatting` | Applies and renders. A true condition with a `background-color` override fills the object, visually confirmed ✓ |
| `ExtendedAttributes` | Not required, not added on paste; omit (§4) ✓ |
| `HideCondition` | Pastes and does not wrongly hide with a false calc. Whether a true calc actually hides the object is untested ○ |
| `ToolTip` | Pastes without error. Neither round-trip survival nor display confirmed ○ |
| `ScriptTriggers` | Pastes without error. Neither round-trip survival nor firing confirmed ○ |

`LocalCSS` on a web viewer is not limited to borders. `background-color`, all four `border-*-radius` corners, and `box-shadow` all apply and render, visually confirmed ✓. The `hover` state is populated on WEBV, so `self:hover .self` is available (§25.4), though the hover transition itself was not observed ◎.

**Containers.** A web viewer nests and pastes attached inside `TabPanelObj`, `PopoverObj` and `SlidePanelObj`, with `Bounds` relative to the container origin exactly as for other object types (§11, §12, §14) ✓ (visually confirmed attached; the return trip was not captured ◎). A single paste containing web viewers inside all three container types alongside fourteen `TextObj`-bearing objects left the trailing Text canary intact and alone, so `ExternalObject` does not trigger the §31 accumulator ✓.

### §15.4 Calculation size and the snippet ceiling

There is **no practical per-calculation length limit**. A single `Calculation` of 60,000 characters round-trips intact, with markup-heavy or plain content alike ✓.

**The constraint is total snippet size.** Above roughly 150 KB, FileMaker pastes every object and silently discards the `Calculation` element from each one. The objects arrive with correct names, bounds and styling, and display nothing. Verified good at 50 KB, 112 KB, 135 KB and 142 KB; verified failing at 516 KB. The threshold was not narrowed further because the behaviour looks like a resource ceiling rather than a designed cap, so a precise figure would not travel between machines or versions.

Generation rule: keep a snippet under about 150 KB. For a large inline payload, split the paste, or deliver the HTML through `Configure Persistent Data` and `Set Web Viewer` (FileMaker 26) rather than inlining it as a `data:` URL.

**Layout saturation.** A layout carrying several dozen live web viewers stops rendering page content in all of them; object-level styling still renders. Not a paste failure, and not quantified. Build web viewer test layouts fresh ◎

---

## §16 ConditionalFormatting

**Round-trip verified.** Conditional formatting survives the clipboard.

Position: **before `Bounds`**, as the first child of the `Object` (FileMaker emits it there on round-trip, ahead of the typed inner element). Sets `Object flags` bit 0 (value 1). ✓

```xml
<Object type="Field" key="1" LabelKey="0" flags="1" rotation="0">
  <ConditionalFormatting>
    <Item id="0" flags="7">
      <Condition op="0">
        <Calculation><![CDATA[Self = "X"]]></Calculation>
        <RangeBegin></RangeBegin>
        <RangeEnd/>
      </Condition>
      <Format>
        <Styles>
          <LocalCSS>
self:normal .self
{
	background-color: rgba(40%,69%,19%,1);
	color: rgba(0%,0%,0%,1);
}
          </LocalCSS>
        </Styles>
      </Format>
    </Item>
  </ConditionalFormatting>
  <Bounds .../>
  <FieldObj ...>...</FieldObj>
</Object>
```

Multiple `<Item>` elements stack correctly in one block — verified with 14 items on a single field, no items dropped or reordered. ✓

The `Item`, `Condition`, `op`, `Calculation` and `RangeBegin` structure round-trips. `Item flags` (§16.1) and `op` (§16.2) are both fully mapped below. ✓

### §16.1 Item flags

Bit 0 is not fill colour — it is the row's own enabled/checked state in the Format > Conditional Formatting dialog:

| Bit | Value | Meaning |
|---|---|---|
| 0 | 1 | Condition enabled (the row's checkbox in the list) ✓ |
| 1 | 2 | Text Color ✓ |
| 2 | 4 | Fill Color ✓ |
| 7 | 128 | Icon Color ✓ |

Additive and enable-gated: `5` = enabled + Fill Color, `7` = enabled + Text Color + Fill Color. A row with only bit 0 set (`flags="1"`) is enabled but has no format property turned on — any `LocalCSS` supplied in that item's `Format` is inert until the corresponding bit is also set. Confirmed directly: an item generated with `flags="1"` plus a populated `background-color` in `LocalCSS` pasted with the Fill Color checkbox unticked and the swatch showing the colour but not applied. ✓

### §16.2 Condition op

| `op` | Meaning |
|---|---|
| 0 | Formula is ✓ |
| 1 | Value is between ✓ |
| 2 | Value is not between ✓ |
| 3 | Value is = (Equal to) ✓ |
| 4 | Value is != (Not equal to) ✓ |
| 5 | Value is > (Greater than) ✓ |
| 6 | Value is < (Less than) ✓ |
| 7 | Value is >= (Greater than or equal to) ✓ |
| 8 | Value is <= (Less than or equal to) ✓ |

### §16.3 Value-op dual storage — a runtime trap

**Value-comparison `Condition` items (op 1–8) store the operand in two places, and both must be generated correctly or the item is dialog-correct but runtime-dead.** `RangeBegin`/`RangeEnd` hold the raw comparison value(s) that the Format > Conditional Formatting dialog reads back — a field with `op="8"` and `RangeBegin>10` displays correctly as "less than or equal to 10" in the dialog. But the actual Browse-mode evaluation runs `Calculation`, which FM materialises as a real formula (e.g. `Self≤10`) — not the operator/value pair. A generated item with an empty or placeholder `Calculation` looks perfectly configured in the dialog and in the XML, yet silently never fires, or always fires (e.g. if `Calculation` is left as a bare constant like `10`, which evaluates as an always-true formula). Confirmed directly: toggling the operator dropdown away and back in the UI forced FM to regenerate the materialised calc from the same `op`/`RangeBegin`, after which the condition began working — proving the dialog state and the runtime state are stored and evaluated separately. ✓

**Generation rule:** for value-comparison conditions, emit all three together — `op`, `RangeBegin`/`RangeEnd`, and a materialised `Calculation` in FM's own comparison form (e.g. `Self≤10`, using the actual `≤`/`≥`/`≠` glyphs FM uses, not ASCII substitutes) — never leave `Calculation` empty or generic when `op` is 1–8.

This closes the earlier open question of whether CF `op=8` survives generation and round-trip structurally with correct dialog rendering — both are confirmed here, along with the runtime trap. See §23 for the failure-mode entry.

---

## §17 ToolTip

Tooltip on hover. Applies to Field, Button, Graphic, and other interactive objects. ✓

```xml
<Object type="Field" key="1" LabelKey="0" flags="0" rotation="0">
  <ToolTip>
    <Calculation><![CDATA["Tick for print on client schedule"]]></Calculation>
  </ToolTip>
  <Bounds top="10" left="10" bottom="30" right="200"/>
  <FieldObj ...>
    ...
  </FieldObj>
</Object>
```

Element order: `ToolTip` comes **after `Bounds`**, before the typed inner element. When `ScriptTriggers` is also present, the full order is `ScriptTriggers`, then `Bounds`, then `ToolTip` (see §21). ✓
Content is a standard `<Calculation>` with CDATA — supports FM expressions. ✓
Sets `Object flags` bit 14 (value 16384). Round-trip verified on fields, buttons, and shapes.
Omit entirely when no tooltip required.

---

## §18 ScriptTriggers

Object-level script triggers. Observed on Field objects; may apply to other types. ✓

```xml
<Object type="Field" key="1" LabelKey="0" flags="0" rotation="0">
  <ScriptTriggers>
    <Trigger event="OnObjectModify" id="3" triggerFlags="1">
      <Script id="682" name="Commit and Refresh"/>
      <TriggerText>Commit and Refresh</TriggerText>
    </Trigger>
  </ScriptTriggers>
  <Bounds top="10" left="10" bottom="30" right="200"/>
  <FieldObj ...>
    ...
  </FieldObj>
</Object>
```

Element order: `ScriptTriggers` comes **before** `Bounds`. ✓

### §18.1 Trigger event IDs

Complete table, all round-trip verified:

| `event` | `id` | Applies to |
|---|---|---|
| `OnObjectEnter` | `1` | Field ✓ |
| `OnObjectExit` | `2` | Field ✓ |
| `OnObjectModify` | `3` | Field ✓ |
| `OnObjectSave` | `4` | Field ✓ |
| `OnObjectKeystroke` | `5` | Field ✓ |
| `OnObjectValidate` | `6` | Field ✓ |
| `OnPanelSwitch` | `7` | TabControl ✓ (also applies to SlideControl by FileMaker design; not captured here) |
| `OnObjectAVPlayerChange` | `8` | Field, TabControl ✓ |

The event vocabulary is object-type dependent. A field carries the enter/exit/modify/save/keystroke/validate set; a tab control carries `OnPanelSwitch`. `OnObjectAVPlayerChange` appeared on both. ✓

`triggerFlags="1"` on all observed instances — meaning unknown, include as-is. ✓
`<Script id name>` reference binds by internal id. It reconnects only if that script id exists in the destination file — pasting into a file without the script leaves the trigger present but pointing at nothing. This is reference rebinding, not a survival failure. ✓
`<TriggerText>` is **derived output, not preserved input**: FM regenerates it on export from the script reference, quote-wrapping the current script name. Verified by generating one trigger with `TriggerText` quoted and one bare in the same paste: with the referenced script id unresolved, both returned identically as `"<unknown>"` (quote-wrapped), so the generated content is discarded and rewritten. When generating, the `TriggerText` content is immaterial; the `<Script id name>` reference is what binds. ✓
Multiple `<Trigger>` elements stack inside one `<ScriptTriggers>` block, in event-id order. ✓
`OnPanelSwitch` also binds when generated on a **SlideControl** (survives round-trip; dangles to `<Script id="0" name="&lt;unknown&gt;"/>` when the id is absent). ✓

### §18.2 Execution contexts — when layout triggers actually fire

Layout and layout-object triggers are a **client focus-model feature**. They fire only when a user interacts with the layout in FileMaker Pro, Go, or WebDirect. They do NOT fire for: Data API / OData writes (no layout objects, no focus, no field exit); imports, Replace Field Contents, or scripts setting fields; server-side and scheduled scripts.

Consequences for generated layouts:
- Do not generate designs whose data integrity depends on `OnObjectValidate` / `OnObjectSave` / `OnObjectModify` — any API write bypasses them silently. Integrity belongs in field validation, auto-enter, and server-side scripts (a Data API call can run a script via its `script` / `script.prerequest` parameters).
- Web-viewer-hosted HTML UIs writing back via the Data API never fire field triggers on the hosting layout. UI logic (skip patterns, conditional hides, cross-field reactions) belongs in the HTML/JS; round-trips to FileMaker go through `FileMaker.PerformScript()` (requires the +64 bit, §15.1) or scripts attached to the API calls.

---

## §19 Button labels, LabelCalc, and icon streams

### §19.1 Standalone button labels are static text

A standalone `Button` object's label is literal text carried as `<Data>` inside the `TextObj`'s `CharacterStyleVector` AND `ParagraphStyleVector`, with a matching `CharacterStyle` — render and round-trip verified on FM 26 (§8). A standalone button does not functionally use `LabelCalc` for its label. Earlier captures showed FM adding an empty `<LabelCalc>` sibling on export as a round-trip artifact; FM 26 copy-backs this cycle did NOT reproduce that (no LabelCalc element on standalone Button or segment exports of Data-labelled buttons) — either way, omit it when generating. ✓

```xml
<TextObj flags="2">
  <Styles>...</Styles>
  <CharacterStyleVector>
    <Style>
      <Data>Save</Data>
      <CharacterStyle mask="32695">...</CharacterStyle>
    </Style>
  </CharacterStyleVector>
  <ParagraphStyleVector>
    <Style>
      <Data>Save</Data>
      <ParagraphStyle mask="0"></ParagraphStyle>
    </Style>
  </ParagraphStyleVector>
</TextObj>
```

### §19.2 LabelCalc is a button-bar-segment feature

`LabelCalc` is the label mechanism for **button-bar segments** — dynamic AND static — not for standalone buttons or PopoverButtons (§14). It is a child of the segment `Object`, positioned **after `ButtonObj`**. When present, the segment's `TextObj` `<Data>` is empty. On FM 26, natively typed static segment labels also serialise as quoted literals in `LabelCalc` (not in `Data`), and generated static `LabelCalc` labels render correctly — see §9/§9.1 for the full mechanism and the Data-label trap. ✓

```xml
<Object type="Button" key="231" ...>   <!-- a button bar segment -->
  <Bounds .../>
  <TextObj flags="2">...<Data/>...</TextObj>
  <ButtonObj buttonFlags="3" iconSize="12" displayType="0"></ButtonObj>
  <LabelCalc>
    <Calculation><![CDATA[2*2]]></Calculation>
  </LabelCalc>
</Object>
```

Supports full FM expressions including conditional logic. Within a bar, a segment either carries a `LabelCalc` or an empty `<Data/>`. ✓

A generated segment `LabelCalc` binds on paste. For a **non-literal expression**, FM sets the segment's `ButtonObj buttonFlags` bit 0 (value 1) as the calculated-label marker and Layout mode displays "Calculation" as the segment label. ✓ For a **quoted literal**, bit 0 is NOT set — native and generated captures both return `buttonFlags="2"` with the literal rendering as the label. ✓

**Invalid calculations are comment-neutralised, not rejected.** A LabelCalc containing an invalid expression pastes successfully with the source wrapped in `/* */` in the calculation dialog — the label is inert and nothing errors anywhere. This is the same neutralisation behaviour Manage Database applies to invalid calcs in pasted table definitions. Validate calculation syntax before generating; FileMaker will not tell you. ✓

(Single observation: an applied-but-neutralised LabelCalc did not re-emit as a `LabelCalc` element on the next copy, while its `buttonFlags` marker remained.)

### §19.3 Button icons are embedded SVG streams

A button icon serialises as binary `<Stream>` blocks inside `ButtonObj`, hex-encoded. Three streams: ✓

| Stream `Type` | Contents |
|---|---|
| `FNAM` | Icon name/font reference |
| `GLPH` | Glyph index (single byte) |
| `SVG ` | The full SVG document, hex-encoded (note the trailing space in the type) |

The SVG stream decodes to a complete standalone SVG (XML declaration, viewBox, path with class `fm_fill`). Because it is self-contained, the icon round-trips intact. ✓

---

## §20 CSS selectors reference

LocalCSS blocks support multiple pseudo-selectors beyond `.self`. All use the `self:normal .selector` prefix pattern.

| Selector | Applies to | Purpose |
|---|---|---|
| `.self` | All objects | Primary object styling ✓ |
| `.text` | Field: renders — confirmed with a margin/padding override, visibly indented on paste ✓. Text object: LocalCSS on `.text` survives and merges into `FullCSS` but has **no visible effect** — confirmed with both a colour override and a padding override, neither rendered on a standalone Text object. Treat as a silent-failure selector on Text objects specifically (see §23); it only has real effect on Field objects. |
| `.icon` | Button, TabPanel | Renders via **`-fm-icon-color`** (and `-fm-icon-padding`), not a plain `color` property. Confirmed by both a native icon-colour change and a generated icon-colour override, both landing correctly when placed in `TextObj > Styles` (see §8) ✓ |
| `.row` | Portal | Default row background — confirmed, survives, merges, renders ✓ |
| `.row_alt` | Portal | Alternating row background — enabled via `-fm-portal-alt-background: true/false` on `.self`, confirmed ✓ |
| `.row_active` | Portal | Active/selected row background — enabled via `-fm-use-portal-current-row-style: true/false` on `.self`, confirmed ✓ |
| `.button_bar_divider` | ButtonBar | Renders via **longhand border properties** (`border-*-color`, `border-*-style`, `border-*-width`), not `background-color`. A generated `background-color` on this selector survives and merges into `FullCSS` but does not visibly change the divider; the divider line is drawn from its border. Confirmed with a 2pt coloured border rendering correctly ✓ |
| `.contents` | Portal, TabPanel | Inner content area — confirmed as a distinct, independently-styled region from `.row`/`.panel`; survives, merges, renders ✓ |
| `.inner_border` | Various | Renders via **longhand border properties only** — the `border:` shorthand is silently stripped and produces no LocalCSS at all on the returned object. Confirmed on a Field with a 2pt coloured longhand border rendering correctly, versus an earlier shorthand attempt that vanished entirely from the returned XML ✓ |
| `.repeat_border` | Field | Repeating field border — generated `LocalCSS` survives, merges into `FullCSS`, and renders ✓ |
| `.baseline` | Text | Bottom border only (underline style) — generated `LocalCSS` survives, merges into `FullCSS`, and renders ✓ |

**Generated LocalCSS that duplicates the active theme's computed value for a given property is pruned from the returned `LocalCSS` on paste** — see §25.3. This explains why some correctly-rendering overrides come back with a shorter `LocalCSS` block than was generated; judge round-trip fidelity by rendered effect, not by byte-identical `LocalCSS` content.

Example — portal with row styling:
```css
self:normal .self
{
	background-color: rgba(100%,100%,100%,1);
}
self:normal .row_active
{
	background-color: rgba(86.3%,94.5%,100%,1);
}
self:normal .row_alt
{
	background-color: rgba(96.9%,97.3%,98.4%,1);
}
```

---

## §21 Object element order

The order FileMaker emits on round-trip.

**Before `Bounds`**, in this order when both are present:
1. `ConditionalFormatting` *(if present)* — a direct `Object` child, before `Bounds`. ✓ (Do NOT place it inside `FieldObj`; a generated object with CF inside `FieldObj` had the CF dropped on paste.)
2. `ScriptTriggers` *(if present)* ✓

Confirmed by round-trip: a field generated with both blocks came back with `ConditionalFormatting` first, `ScriptTriggers` second, both still ahead of `Bounds`. ✓

**Then:**
1. `Bounds`
2. `ToolTip` *(if present)* — after `Bounds`, before the typed inner element ✓
3. Typed inner element (`FieldObj`, `TextObj`, `ButtonObj`, `RectObj`, etc.). For a `FieldObj` the internal order is: `Name`, `ExtendedAttributes`, `Styles`, `PlaceholderText` *(if present)*, `DDRInfo` (the latter carries any `ValueList` descriptor). ✓
4. `RenderFormat` *(shapes only — but note FM emits `HideCondition` BEFORE `RenderFormat` on shapes; either order pastes, FM normalises)* ✓
5. `CanEntryCalc` *(if present, fields only — FileMaker 2026)* ✓
6. `HideCondition` *(if present)* ✓

**Trailing children.** `CanEntryCalc` and `HideCondition` are the LAST children of the `Object`, after the entire typed inner element block (and after `RenderFormat` on shapes). When both are present, FileMaker normalises them to **`CanEntryCalc` then `HideCondition`**, regardless of paste order. ✓

**Scope.** `CanEntryCalc` is for fields only — on a non-field object it can cause the whole object to fail to paste (§27), so never attach it elsewhere. `HideCondition` applies to any object type, including shapes, where FM emits it before `RenderFormat`. ✓

**Object flags set by behavioural elements** (do not set these when generating; FileMaker sets them from object state):
- bit 0 (1) — has `ConditionalFormatting` ✓
- bit 2 (4) — has `HideCondition` ✓
- bit 14 (16384) — has `ToolTip` ✓ (isolated from a tooltip-only-among-before-Bounds field). Whether `PlaceholderText` alone also sets this bit was not isolated; `PlaceholderText` definitely sets `FieldObj` bit 17 (131072), which is a separate flag.

For container types (ButtonBar, TabControl, Portal, PopoverButton, SlideControl, Group), nested child `Object` elements are inside the typed inner element, after that element's `Styles`.

---

## §22 Step reference

```xml
<Step enable="True" id="1" name="Perform Script">
  <CurrentScript value="Pause"/>
  <Script id="257" name="ScriptName"/>
</Step>

<!-- Go to Layout -->
<Step enable="True" id="6" name="Go to Layout">
  <LayoutDestination value="SelectedLayout"/>
  <Layout id="34" name="LayoutName"/>
</Step>

<!-- Go to Related Record -->
<Step enable="True" id="..." name="Go to Related Record">
  <Option state="False"/>
  <MatchAllRecords state="False"/>
  <ShowInNewWindow state="False"/>
  <Restore state="True"/>
  <LayoutDestination value="SelectedLayout"/>
  <NewWndStyles Style="Document" Close="Yes" Minimize="Yes"
                Maximize="Yes" Resize="Yes" Styles="3606018"/>
  <Table id="1065146" name="TableOccurrenceName"/>
  <Layout id="34" name="LayoutName"/>
</Step>

<!-- Go to Record/Request/Page -->
<Step enable="True" id="..." name="Go to Record/Request/Page">
  <NoInteract state="False"/>
  <RowPageLocation value="First"/>
</Step>

<!-- Enter Find Mode -->
<Step enable="True" id="..." name="Enter Find Mode">
  <Pause state="True"/>
  <Restore state="False"/>
</Step>

<!-- Delete Portal Row -->
<Step enable="True" id="..." name="Delete Portal Row">
  <NoInteract state="True"/>
</Step>

<!-- Delete Record/Request -->
<Step enable="True" id="..." name="Delete Record/Request">
  <NoInteract state="False"/>
</Step>

<!-- Export Field Contents -->
<Step enable="True" id="..." name="Export Field Contents">
  <CreateDirectories state="False"/>
  <AutoOpen state="False"/>
  <CreateEmail state="False"/>
  <Field table="TableName" id="164" name="FieldName"/>
</Step>

<!-- No children -->
<Step enable="True" id="..." name="Show All Records"/>
<Step enable="True" id="..." name="New Record/Request"/>
<Step enable="True" id="..." name="Delete All Records">
  <NoInteract state="False"/>
</Step>
```

Round-trip behaviour of button steps (FM Pro 26):
- Parameterised steps survive generation intact — `Go to Record/Request/Page` verified with `NoInteract` and `RowPageLocation` (`First` and `Last`). ✓
- FM adds `<DisableStepCollapsed state="False"/>` as a step child on round-trip, and `<CurrentScript value="Pause"/>` to Perform Script — artifacts, not required when generating. ✓
- Script references bind **by id only**; an id not present in the file returns `<Script id="0" name="&lt;unknown&gt;"/>` — no rebinding by name. ✓
- `<Step id="0" name="None"/>` does not persist (`ButtonObj` returns empty) — harmless; omit it for a no-action button. ✓

---

## §23 Silent failure modes

- `type="FMObjectList"` instead of `"LayoutObjectList"` — entire paste dropped silently ✓
- Tabs instead of spaces — elements dropped silently ✓
- Missing required Step children — step parameters dropped silently ✓
- Unknown `ThemeName` — FM substitutes the file's default theme, both at render and in the returned XML (no poisoned identifier survives onward) ✓
- Invalid calculation inside a calc element (confirmed for `LabelCalc`) — comment-neutralised to `/*…*/`, object pastes, calc inert, nothing errors (§19.2) ✓
- Dynamic (non-literal) `TitleCalc` on a non-front tab panel — dropped when every batch `TextObj` carries `ExtendedAttributes`; migrated into the next EA-less `TextObj` otherwise (§11, §31) ✓
- Web viewer generated without the +64 `externalFlagSet` bit — `FileMaker.PerformScript()` is defined, returns normally and throws nothing, but the call never reaches FileMaker; no error on either side (§15.1) ✓
- Snippet larger than roughly 150 KB — every `Calculation` element is dropped from the paste while the objects themselves arrive intact, correctly named and styled, showing nothing (§15.3) ✓
- Button/segment-level LocalCSS placed in an object-level `Styles` block instead of `TextObj > Styles` — pastes without error, produces no visual effect, absent from the returned XML entirely (§8) ✓
- `.text` selector overrides on Text objects (as opposed to Field objects) — survive, merge into `FullCSS`, produce no visible effect (§20) ✓
- `.button_bar_divider` and `.inner_border` styled with non-border properties, or with the `border:` shorthand — survive-or-vanish with no visible effect (§20) ✓
- Value-comparison Conditional Formatting items (`op` 1–8) with an unmaterialised `Calculation` — the Format dialog shows a fully correct, ticked condition; only a Browse-mode behaviour check reveals it never evaluates correctly. Undetectable by structural audit or by opening the dialog (§16.3) ✓
- `CustomStyles > Name` referencing a style absent from the destination theme — the object pastes, renders its base appearance, and returns with no `CustomStyles` block at all; nothing errors (§25.3) ✓
- Web viewer generated with `+4`/`+8` (`externalFlagSet` `32781`) at a small object size — progress bar and status chrome consume the full height, the page loads but nothing is visible (§15.1) ✓
- A single selected Text object copying to the system clipboard as plain text instead of layout XML — no error shown, affects any single-object capture workflow (§0) ✓

---

## §24 Generation defaults

```
Object flags:     "0"
FieldObj flags:   "0"  (or "32" enterable portal / "36" non-enterable)
rotation:         "0"
LabelKey:         "0"
key:              any integer — FM reassigns
portalFlags:      "21" (scrollbar, no sort, no filter)
initialRow:       "1"
ThemeName:        match the target file's theme
externalFlagSet:  "32865" (WebViewer, bridge-enabled — see §15.1 for the bit table; use "32781" for plain URL display)
Anchoring:        "0" = left+top (default). "-1073741824" = all four (full-bleed). See §2.2
```

Do not generate Object flags bits 0, 2, 3, 8, 9, 12, 13, 14, 16, 24 or 25. FileMaker sets them from object state, or they do not exist (§2.1).

Bits 28 to 31 are the exception: they are object anchoring and **should** be generated when an object must resize with the window. `flags="0"` is left and top anchored, the FileMaker default. Use `-1073741824` for a full-bleed object. Full table in §2.2.

**Always include `ExtendedAttributes` on generated `TextObj`s** (Text and Button objects) — see §31. This is the one exception to the "omit round-trip artifacts" guidance in §4; leaving it out is the root cause of multi-object paste corruption.

---

## §25 Object styling and serialization model

This is the core of how a styled object is written out. It is **theme-independent** — the structure below is identical regardless of which theme is active. Only the values differ between themes (see §27).

### §25.1 The three CSS elements

Inside `<Styles>`, an object carries up to three CSS blocks plus the theme name:

- **`<LocalCSS>`** — the *changed-property delta only*, grouped by state. Present when the object overrides its theme defaults.
- **`<CustomStyles><Name>…</Name></CustomStyles>** — a reference to a named theme style. `Name` carries the style's identifier as the theme stores it. **User-created styles are keyed by their display name** and bind from that name alone. Stock-theme styles may instead carry an `FM-`UUID (e.g. `FM-11711CFC-75AA-486A-B945-C847FEF44E34`); use whichever form the theme holds. Either way the reference is a pointer: the style's appearance lives in the theme, not in the `CustomStyles` block. ✓
- **`<FullCSS>`** — the *full computed merge*: every property, the complete resolved appearance. FileMaker ALWAYS recomputes this from the destination theme on paste.
- **`<ThemeName>`** — closes the `Styles` block.

### §25.2 The four cases

| Case | Elements emitted, in order |
|---|---|
| Theme default (no override) | `FullCSS`, `ThemeName` |
| Local override | `LocalCSS`, `FullCSS`, `ThemeName` |
| Named style applied | `CustomStyles`, `FullCSS`, `ThemeName` |
| Named style + local override | `LocalCSS`, `CustomStyles`, `FullCSS`, `ThemeName` |

The two `CustomStyles` cases are generatable: emit the style's identifier in `<CustomStyles><Name>` and it binds on paste (see §25.3). The identifier must exist in the target theme.

### §25.3 Generation rule

When generating, emit a minimal `FullCSS` (the handful of properties you care about) plus `ThemeName`. **FileMaker recomputes `FullCSS` from the destination theme's tokens on paste**, expanding a minimal block to the full baseline. For deliberate overrides, also emit a `LocalCSS` delta containing only the changed properties grouped under the relevant `self:STATE .selector` heading. Do not hand-write the full baseline — let FileMaker compute it.

**Applying a named style by generation works. ✓** Emit `<CustomStyles><Name>STYLE-NAME</Name></CustomStyles>` plus `ThemeName`, and on paste the object binds to that style and renders it. The binding survives a subsequent copy. Verified end to end on WebViewer objects: two user-created styles generated from their display names alone, no exemplar object, both bound and returned their real computed appearance.

**Scope.** Verified on `ExternalObject`. `Styles` is a shared block across object types, so the mechanism is expected to hold generally, but binding by display name has not been captured on Field, Text or Button objects, and whether a theme style is scoped to the object type it was created on is untested. ◎

**Ask the user for the style name.** For user-created styles the display name shown in the theme is the identifier, so a name is all a generator needs. Stock-theme styles may use an `FM-`UUID instead; that form can only be sourced from an object that already carries the style, since no catalogue of theme styles is exposed through the clipboard format.

**A `CustomStyles` name that does not exist in the target theme is dropped silently** (§23). The object pastes, renders its base appearance, and returns with no `CustomStyles` block at all. Nothing errors. This is also what an identifier from a different file looks like, so verify the style exists in the destination before relying on it.

**Delta-pruning on return.** FM prunes generated `LocalCSS` declarations that are redundant against the active theme's computed value before returning them on copy. Confirmed on a ButtonBar divider: a generated `border-*-style: solid` (matching the theme's own default divider style) was silently dropped from the returned `LocalCSS`, while the non-default `border-*-color` and `border-*-width` in the same block survived. This is not a failure — the declaration still applied at paste time — it is evidence that `LocalCSS` is stored/returned as a true delta against the theme, not as a literal echo of what was generated. When auditing round-trip fidelity, judge by rendered effect, not by byte-identical `LocalCSS` content. ✓

### §25.4 State vocabulary

`normal`, `hover`, `pressed`, `focus`, `checked`, `checkedfocus`, `placeholder`, `droptarget`. Each appears as a `self:STATE .selector { … }` block. Which states a theme actually populates is theme-dependent — a minimal theme may emit only `normal`, `focus`, and `placeholder` for a field, while a richer theme adds `hover` and `droptarget`. The vocabulary is universal; the populated subset is not. ✓

### §25.5 Fill and gradient

- Solid fill emits `background-color` together with `background-image: none` and `border-image-source: none`. ✓
- Gradient fill emits `-webkit-gradient(...)` in `background-image` with a transparent `background-color`. Radial, linear, and multi-stop (`color-stop(0.5,...)`) all round-trip. ✓

### §25.6 Retheming objects to a named style

Because a named style applies by identifier (§25.3), an object's style can be reassigned by rewriting its `<CustomStyles><Name>`, and an overridden object can be put onto a style by removing its `<LocalCSS>` and inserting a `<CustomStyles>` name. Verified end to end: a set of fields each carrying a `font-size` `LocalCSS` override, restyled by stripping the override and adding a style id, pasted clean and on-style. ✓

Method:
1. Identify the target objects. "Overridden" objects are those with a `<LocalCSS>` block (the local-override and named-style-plus-override cases of §25.2).
2. For each, drop the `<LocalCSS>` block and replace the `<Styles>` contents with `<CustomStyles><Name>STYLE-NAME</Name></CustomStyles>` and the `<ThemeName>`.
3. For a user-created style the display name is the identifier. For a stock-theme style carrying an `FM-`UUID, source it from an object that already has the style — see §25.3.

Paste workflow (matters — these are paste mechanics, not XML):
- **Emit only the objects being restyled.** A retheme snippet pasted onto a layout that still contains the originals merges with them; text objects in particular concatenate their content. Delete the originals, then paste the replacements.
- **Use Paste in Place** so objects return to their stored `Bounds`. A plain paste drops them at a cursor offset; the position is in the XML either way, but only Paste in Place honours it.
- Clear `LabelKey` to `0` on restyled fields if their label objects are not included in the snippet.

**Open caution (untested).** Replacing objects wholesale gives them new internal keys. References that bind by key — a label's `LabelKey` to its field, tab order, button actions targeting an object — may not reattach to the replacements. Not tested. Before relying on object-replacement retheme for a layout where fields have attached labels or a defined tab order, verify those associations survive, or retheme by editing objects in place rather than replacing them.

---

## §26 Character-attribute Face bitmask

The `Face` integer in a `CharacterStyle` is an additive bitmask. Every common bit below is round-trip verified from hand-built fields. ✓

| Bit | Value | Attribute |
|---|---|---|
| 2^0 | 1 | strikethrough |
| 2^1 | 2 | small-caps |
| 2^2 | 4 | superscript |
| 2^3 | 8 | subscript |
| 2^4 | 16 | uppercase |
| 2^5 | 32 | lowercase |
| 2^6 | 64 | Word Underline (underlines words, not spaces) ✓ |
| 2^7 | 128 | double-underline |
| 2^8 | 256 | bold |
| 2^9 | 512 | italic |
| 2^10 | 1024 | underline (single, continuous) |
| 2^12 | 4096 | highlight |
| 2^13 | 8192 | condensed |
| 2^14 | 16384 | expanded |

Notes:
- **Case transform is a two-bit field.** uppercase = 16, lowercase = 32, capitalize = both (48), none = 0. ✓
- **Underline is a three-state field across three separate bits, not independent toggles.** Word Underline (64), double underline (128), and single/continuous underline (1024) are three distinct visual styles corresponding to the "Underline / Word Underline / Double Underline" options in FileMaker's text style picker. Confirmed by isolated round-trip: bit 64 alone renders as underline that skips spaces between words, distinct from bit 1024's continuous underline. Combining multiple underline bits at once is untested — generate only one of the three at a time. ✓
- Bold and italic are coordinated three-part changes. Each sets its Face bit AND the CSS (`font-weight: bold` / `font-style: italic`) AND swaps the postscript font-family variant (e.g. `HelveticaNeue-Bold`, `HelveticaNeue-Italic`). Generate all three together. ✓
- Underline and strikethrough also travel as CSS (`-fm-underline: underline` | `double-underline`, `-fm-strikethrough: true`) alongside their Face bits. ✓
- **Bits 2^11 (2048) and 2^15 (32768) are confirmed inert** — isolated round-trip on each produced no visible effect of any kind, in isolation and in combination with bit 64. Safe to treat as unused/reserved rather than merely unobserved. ✓

### §26.1 Confirmed CSS character/paragraph vocabulary

All round-trip verified in `LocalCSS`: per-side border colour/width/style, dashed/dotted, arbitrary radius; `box-shadow`; radial/linear/multi-stop gradients; multi-state stacks; `text-align`; `-fm-text-vertical-align`; `font-style: italic`; `text-transform` uppercase/lowercase/capitalize; `font-variant: small-caps`; `font-stretch` condensed/expanded; `-fm-underline` underline/double-underline; `-fm-strikethrough`; `-fm-glyph-variant` superscript/subscript; `-fm-highlight-color`; `line-height`; `font-size`; `color`; `direction: rtl`; `-fm-tategaki`; `-fm-fill-effect`; `-fm-borders-baseline`. ✓

---

## §27 FileMaker 2026: access-by-calculation (CanEntryCalc)

FileMaker 2026 added calculation-driven control over object access states. The field-entry state serialises as `<CanEntryCalc>`. ✓

```xml
<Object type="Field" key="1" flags="0" rotation="0">
  <Bounds .../>
  <FieldObj flags="1048608" ...>
    ...
    <Styles>...</Styles>
    <DDRInfo>...</DDRInfo>
  </FieldObj>
  <CanEntryCalc>
    <Calculation><![CDATA[Get ( AccountPrivilegeSetName ) = "Admin"]]></Calculation>
  </CanEntryCalc>
</Object>
```

- Position: LAST `Object` child, same slot as `HideCondition`. With both present: `CanEntryCalc` then `HideCondition`. ✓
- Scope: **fields only.** On a non-field object it is unsafe — a generated rectangle carrying `CanEntryCalc` failed to paste at all (the whole object was dropped, not just the element). Never attach it to anything but a field. ✓
- Contains a standard `<Calculation>` with CDATA. ✓
- A generated `CanEntryCalc` enforces on paste (a true calc allows entry, a false calc blocks it). Base `FieldObj` flags are fine; the high access flag bits a natively built field carries are not required for the calc to take effect. ✓
- **The Object-level flag bits governing which access mode is active (bits 2/24 for Browse mode, bits 4/25 for Find mode) are documented in full in §5.2.1.** `CanEntryCalc` itself is a single, shared calculation element referenced by whichever bit-pair(s) are set to Set by Calculation — there is one calc per field, not one per mode. ✓

**Closed.** A full sweep of the Field entry-behavior dialog (Edit / Select Only / View Only / Set by Calculation, in both Browse and Find modes) produced no new calc-driven element beyond `CanEntryCalc`. It is confirmed as the sole calc-driven access mechanism on FM Pro 26.0.1.51.

---

## §28 Theme independence

The serialization structure in §25–§27 is identical across themes. Verified by building the same objects under stock `apex_blue` and a custom minimalist theme: same element shapes, same ordering, same selector set, same property list, same `LocalCSS`/`FullCSS` model. Only the values move. ✓

What is theme-specific (recomputed from the destination theme on paste):
- All colours and `rgba()` values in `FullCSS`
- Border radius, font family/size defaults, padding/margins
- The set of state blocks actually populated
- The named-style palette (each theme ships its own roster)

### §28.1 ThemeName format

- **Stock theme:** `com.filemaker.theme.{name}` — e.g. `com.filemaker.theme.apex_blue`
- **Custom theme:** `com.filemaker.theme.custom.{UUID}` — e.g. `com.filemaker.theme.custom.AE789D5E_9720_433C_B2B0_498EB8D684D4`

Saving a change to a stock theme forks it into a custom theme with a UUID-suffixed id under the `.custom.` namespace. Renaming the theme is cosmetic (display name only); the internal id stays the UUID. Generated objects must carry the exact destination `ThemeName` id verbatim, or the paste will not bind to the right theme. ✓

---

## §29 What round-trips, and what does not

Object-level definition survives the round trip. If something appears to drop, suspect the format, not the clipboard.

Confirmed to survive: all styling (fill, gradient, borders, shadows, the full CSS vocabulary, the Face bitmask), structure, conditional formatting, tooltip, hide condition, placeholder, script triggers, button icons (embedded SVG streams), button-bar `LabelCalc`, and the 2026 `CanEntryCalc`.

Does NOT travel with a copied object, by nature rather than by format error:
- **Record-bound content** — a field shows record data, not stored layout text; there is nothing to carry.
- **Cross-file references that cannot rebind** — script-trigger and value-list references bind by internal id; they reconnect only if that id exists in the destination file. Present but unresolved otherwise. This is reference resolution, not a survival failure.
- **Theme-level and layout-level properties** — part styling, layout background, the full theme palette, theme colour swatches, default-style designation. These are not object properties, so a copied object does not carry them. They require Save as XML (see §30).

### §29.1 Value list attachment

A value-list binding travels in two parts: a `<ValueList>NAME</ValueList>` child of `FieldObj` and a `<ValueList name="NAME" id="N"/>` descriptor in `DDRInfo` (full form in §5.3). Both must be present; a `DDRInfo` descriptor on its own drops on paste. The `id` binds the list and is a small integer sourced from a field already using that list. A generated field carrying both forms attaches the value list on paste (verified, `displayType` 1 and 2). ✓

---

## §30 Out of scope (separate Save as XML project)

This spec covers the object-level clipboard format (`fmxmlsnippet type="LayoutObjectList"`). The following are theme-level or layout-level and are NOT carried by the clipboard. They require Save as XML and are a separate body of work, not gaps in this spec:

- Layout part styling (header/body/footer/sub-summary fills, alternating-row backgrounds)
- Layout background
- The full named-style palette per theme (only individually referenced styles come through the clipboard)
- Theme colour swatches and default-style designation

### §30.1 Not yet verified

- `TextObj flags="2"` observed on a single Text object where every sibling in the same paste carried `0`. Sits on the inner element, not `Object`, so unrelated to anchoring. Not yet reproduced against a known Inspector setting. ○

---

## §31 Multi-object Text paste corruption

Pasting two or more Text objects in one operation without an `ExtendedAttributes` block on each `TextObj` causes FileMaker's paste handler to concatenate each object's `Data` onto every one that follows it. Including a standard `ExtendedAttributes` block on each object — mirroring the object's own `CharacterStyle` — eliminates the corruption entirely. Confirmed at both n=2 and n=3 objects in one paste, all independent and clean. ✓

**Always generate `ExtendedAttributes` on `TextObj`s** (Text and Button objects) — this is the one exception to the round-trip-artifact omission list in §4.

```xml
<TextObj flags="0">
  <ExtendedAttributes fontHeight="10" graphicFormat="0">
    <NumFormat flags="0" charStyle="0" negativeStyle="0" currencySymbol="" thousandsSep="0" decimalPoint="0" negativeColor="#0" decimalDigits="0" trueString="" falseString=""/>
    <DateFormat format="0" charStyle="0" monthStyle="0" dayStyle="0" separator="0">
      <DateElement>0</DateElement>
      <DateElement>0</DateElement>
      <DateElement>0</DateElement>
      <DateElement>0</DateElement>
      <DateElementSep index="0"/>
      <DateElementSep index="1"/>
      <DateElementSep index="2"/>
      <DateElementSep index="3"/>
      <DateElementSep index="4"/>
    </DateFormat>
    <TimeFormat flags="0" charStyle="0" hourStyle="0" minsecStyle="0" separator="0" amString="" pmString="" ampmString=""/>
    <CharacterStyle mask="32695">
      <Font-family codeSet="Other" fontId="0" postScript="HelveticaNeue">Helvetica Neue</Font-family>
      <Font-size>16</Font-size>
      <Face>0</Face>
      <Color>#282828</Color>
    </CharacterStyle>
  </ExtendedAttributes>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
  <CharacterStyleVector>
    <Style>
      <Data>Label text</Data>
      <CharacterStyle mask="32695">...</CharacterStyle>
    </Style>
  </CharacterStyleVector>
  <ParagraphStyleVector>
    <Style>
      <Data>Label text</Data>
      <ParagraphStyle mask="0"></ParagraphStyle>
    </Style>
  </ParagraphStyleVector>
</TextObj>
```

The `ExtendedAttributes > CharacterStyle` values should mirror the object's own `CharacterStyleVector > Style > CharacterStyle` values. `NumFormat`/`DateFormat`/`TimeFormat` can stay at the placeholder defaults shown — they don't need to be meaningful for a Text object, only present.

### §31.1 Scope

**Covered by `ExtendedAttributes` on every `TextObj`:** standalone `Text` objects (n=2, n=3), `Button` objects, `ButtonBar` segments, and `GroupButton` child Texts — all clean in large mixed batches (16 and 15 top-level objects, FM Pro 26). ✓

**Confirmed participants beyond `TextObj` labels:**
- **Dynamic (non-literal) `TitleCalc` on a non-front `TabPanel`** feeds the accumulator: without EA downstream its calc source migrates into the next EA-less `TextObj` (observed landing on a PopoverButton, destroying its label); with EA everywhere it is silently dropped instead. Generate dynamic titles only as quoted literals on non-front panels (§11). ✓
- **PopoverButton `TextObj`s must carry `ExtendedAttributes`** — a bare `<TextObj flags="0"/>` was the leak absorber in testing. ✓

- **Fields carrying `PlaceholderText` (§5.6)** feed the accumulator via the placeholder's `Calculation`, and are covered by the same sink-side fix: with `ExtendedAttributes` on every `TextObj`, multiple placeholder fields batch freely (n=2 round-tripped, n=4 interleaved with Texts and a Button). `ExtendedAttributes` on the `FieldObj` is inert for this purpose. The placeholder source survives rather than being dropped, unlike `TitleCalc`. No per-paste limit applies. ✓

### §31.2 Scope of the fix

`ExtendedAttributes` is the only variable that prevents the corruption. `CharacterStyle mask`, font, size, colour, `Face`, and spatial distance between objects make no difference on their own. ✓

`ExtendedAttributes` on every internal `TextObj` covers all container cases, including the worst sequence found: a three-segment `ButtonBar` immediately followed by a `GroupButton` with a Text child. `TabControl` panel titles are `TitleCalc`-based rather than `TextObj` and are unaffected either way. ✓

### §31.3 Generation rule

Include `ExtendedAttributes` on every generated `TextObj` — Text, Button, ButtonBar segments, GroupButton children, and PopoverButtons — even when only one is being pasted; it costs nothing and removes the risk. Mixed batches of all of these are confirmed clean with the block present, and the rule also covers `PlaceholderText` fields, which need nothing extra and batch freely (§5.6). ✓

Confirmed at scale: 14 `TextObj`-bearing objects interleaved with 13 WebViewers in a single paste, all clean. `ExternalObject` does not participate in the accumulator in either direction. ✓

