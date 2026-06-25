# FileMaker Layout XML Spec

Paste-ready FileMaker layout object XML (`fmxmlsnippet type="LayoutObjectList"`), empirically derived from round-trip testing across multiple applications and themes.

**✓** = round-trip verified  **◎** = observed across multiple layouts  **○** = single-observation hypothesis

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
- Never default to `com.filemaker.theme.apex_blue` unless confirmed. The examples in this spec use `apex_blue` as a placeholder only.
- Use the extracted identifier verbatim in every `<ThemeName>` element throughout the generated XML.

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
| `flags` | **Use `0` for generation.** See §2.1 |
| `rotation` | Tenths of degrees. `0` = no rotation. `900` = 90°. ✓ |
| `name` | Optional. Direct layout object name (WebViewers, named ButtonBars) ◎ |

### §2.1 Object flags — generation rule

**Use `flags="0"` for all generated objects.** FM sets these from object state. The table below documents observed values for analysis purposes — do not set these when generating.

| Bit | Value | Meaning |
|---|---|---|
| 0 | 1 | Has `ConditionalFormatting` ✓ |
| 2 | 4 | Object has a HideCondition ✓ |
| 3 | 8 | Portal field row option ◎ |
| 8 | 256 | Object has icon (ButtonObj icon streams present) ◎ |
| 9 | 512 | Layout part marker ○ |
| 12 | 4096 | Line: print-only visibility ○ |
| 13 | 8192 | Line: screen-only visibility ○ |
| 14 | 16384 | Has `ToolTip` ✓. (Placeholder presence is a separate `FieldObj` flag, bit 17, not this Object bit.) |
| 16 | 65536 | Named layout object ◎ |
| 24 | 16777216 | Field access-state marker / input mode ◎ (see §27; appears on 2026 fields carrying access states) |
| 25 | 33554432 | Field access-state marker ○ (see §27) |
| 28 | 268435456 | WebDirect/rendering tier flag (set by FM, safe to omit) ◎ |
| 29 | 536870912 | WebDirect/rendering tier flag (set by FM, safe to omit) ◎ |
| 30 | 1073741824 | WebDirect/rendering tier flag (set by FM, safe to omit) ◎ |
| 31 | -2147483648 | Locked in layout mode ◎ |

The generation rule is simple: use `flags="0"` and let FileMaker set these.

**Bits 28, 29, 30** appear on nearly every object in WebDirect-optimised layouts — set at the layout/theme level by FM, not per-object by the developer. Safe to omit when generating; FM will apply as needed.

**Bit 24** appears on fields across many display types including edit boxes, calendars, and drop-downs. Likely a field-level input mode setting, not display-type specific.

Common production values (do not generate — FM sets these):

| Value | Bits | Context |
|---|---|---|
| `260` | 2,8 | Standard nav ButtonBar segment |
| `261` | 0,2,8 | Active nav ButtonBar segment |
| `65544` | 3,16 | Named ButtonBar segment |
| `65545` | 0,3,16 | Named active ButtonBar segment |
| `-2147483648` | 31 | Locked object |
| `16777216` | 24 | Field with touch input flag |
| `268435456` | 28 | WebDirect layout object |
| `805306368` | 28,29 | WebDirect layout object (tier 2) |

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
| `RRect` | *(none)* | Requires `RenderFormat` ◎ |
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
- `ExtendedAttributes` — FM generates from field type and formatting settings ✓
- `DDRInfo` — FM populates from the file's own field registry ✓
- `ParagraphStyleVector` — FM adds on export; not required for paste ✓
- `SlidePanel > Styles` — FM adds on export; not required for paste ✓

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
| `numOfReps` | `1` | Repetitions to display |
| `flags` | `0` | See §5.2 |
| `inputMode` | `0` | Input method |
| `keyboardType` | `1` | Touch keyboard type |
| `displayType` | `0` | Control style — see §5.3 |
| `quickFind` | `1` | `0` = excluded. Mirrors flags bit 15 |
| `pictFormat` | `5` | Container display format |

### §5.2 FieldObj flags

| Bit | Value | Meaning |
|---|---|---|
| 0 | 1 | Include other value (radio/checkbox sets) ○ |
| 2 | 4 | Not enterable in Browse mode ✓ |
| 5 | 32 | Tab to next object ✓ |
| 10 | 1024 | Calendar popup button (with bit 19) ✓ |
| 11 | 2048 | Auto-complete using existing values ○ |
| 15 | 32768 | Quick Find off — also sets `quickFind="0"` ✓ |
| 19 | 524288 | Calendar popup button (with bit 10) ✓ |
| 20 | 1048576 | Edit box marker — set when displayType=0 ✓ |

Common combinations:
- `0` — default
- `32` — tab only
- `36` — not enterable + tab ◎
- `32804` — not enterable + tab + Quick Find off ✓
- `32800` — tab + Quick Find off ✓
- `525344` — tab + calendar button (bits 5,10,19) ✓
- `1048608` — tab + edit box marker (bits 5,20) ✓

### §5.3 FieldObj displayType

| Value | Control |
|---|---|
| `0` | Edit box ✓ |
| `1` | Drop-down list ✓ |
| `2` | Pop-up menu ✓ |
| `3` | Checkbox set ◎ |
| `4` | Radio button set ◎ |
| `5` | Unobserved — may not exist |
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

### §5.6 Portal field bounds

Fields inside a portal use **relative** bounds. First data row starts at `top="4"`. ◎  
Header-row fields use `top="-1"` to sit above the scrolling area. ◎

---

## §6 Text

```xml
<Object type="Text" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="10" left="10" bottom="25" right="200"/>
  <TextObj flags="0">
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
  <TextObj flags="0">
    <Styles>
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
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
</Object>
```

Button label text lives in `TextObj > CharacterStyleVector > Style > Data`. `TextObj` is required on buttons. `LabelCalc` is ignored for static labels — do not use it. ✓

### §8.1 ButtonObj attributes

| Attribute | Values |
|---|---|
| `buttonFlags` | `0` = no toggle; `2` = toggle; `3` = toggle + option |
| `iconSize` | `0`–`19` |
| `displayType` | `0`–`4` (text/icon display mode) |

### §8.2 Layout object name

When named via "Set Object Name", stored at `Object > TextObj > Styles > CustomStyles > Name`:

```xml
<TextObj>
  <Styles>
    <CustomStyles>
      <Name>FM-UUID-GOES-HERE</Name>
    </CustomStyles>
    <ThemeName>...</ThemeName>
  </Styles>
</TextObj>
```

Triggers `Object flags` bit 16. ◎

### §8.3 HideCondition

`HideCondition` is the LAST child of the `Object`, after the entire typed inner element block (and after `RenderFormat` on shapes). Applies to any object type. When `CanEntryCalc` is also present, the order is `CanEntryCalc` then `HideCondition`. See §21. ✓

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

```xml
<Object type="ButtonBar" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="0" left="0" bottom="35" right="300"/>
  <ButtonBarObj flags="0" segmentKey="0">
    <Object type="Button" key="2" LabelKey="0" flags="260" rotation="0">
      <Bounds top="1" left="1" bottom="34" right="150"/>
      <TextObj flags="2">
        <Styles>
          <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
        </Styles>
        <CharacterStyleVector>
          <Style>
            <Data>Home</Data>
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
        <Step enable="True" id="0" name="None"/>
      </ButtonObj>
    </Object>
    <Object type="Button" key="3" LabelKey="0" flags="260" rotation="0">
      <Bounds top="1" left="150" bottom="34" right="299"/>
      <TextObj flags="2">
        <Styles>
          <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
        </Styles>
        <CharacterStyleVector>
          <Style>
            <Data>Detail</Data>
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
        <Step enable="True" id="0" name="None"/>
      </ButtonObj>
    </Object>
  </ButtonBarObj>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
</Object>
```

- `ButtonBarObj` requires `flags="0" segmentKey="0"` attributes ✓
- Button label text in `TextObj > CharacterStyleVector > Style > Data` ✓
- `TextObj flags="2"` inside ButtonBar segments (not `"0"`) ✓
- Button segment bounds start at `(1,1)` not `(0,0)` — FM adds 1pt inset ✓
- Button segments are adjacent: second button's `left` = first button's `right` ✓
- `LabelCalc` is ignored — do not use ✓

**Button Object flags in ButtonBar:**

| Value | Bits | Use |
|---|---|---|
| `260` | 2,8 | Standard segment |
| `261` | 0,2,8 | Active or icon-only segment |
| `256` | 8 | Single-segment bar |
| `65544` | 3,16 | Named segment |
| `65545` | 0,3,16 | Named active segment |

Bit 0 = currently active layout's button — FM sets this on save. Use `260` for standard, `261` for icon-only when generating. ◎

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

| Bit | Value | Meaning |
|---|---|---|
| 0 | 1 | Scrollbar ◎ |
| 2 | 4 | Alternating row colours ○ |
| 3 | 8 | Sort enabled ◎ |
| 4 | 16 | Required base flag — always present ◎ |
| 5 | 32 | Unknown ○ |
| 7 | 128 | Filter enabled ◎ |
| 8 | 256 | Allow deletion of portal records ○ |

| Value | Bits | Scenario |
|---|---|---|
| `16` | 4 | No sort, no filter, no scrollbar |
| `17` | 0,4 | No sort, no filter, scrollbar only |
| `21` | 0,2,4 | Scrollbar + alternating rows |
| `25` | 0,3,4 | Sort, no filter, scrollbar |
| `56` | 3,4,5 | Sort + unknown bit 5 ○ |
| `145` | 0,4,7 | No sort, filter, scrollbar |
| `149` | 0,2,4,7 | Filter + alternating rows |
| `153` | 0,3,4,7 | Sort + filter |
| `157` | 0,2,3,4,7 | Sort + filter + alternating rows |
| `401` | 0,4,7,8 | Planner/single-row display ○ |

### §10.3 SortList

Empty (required even with no sort): `<SortList>
</SortList>` ✓

Sort structure (requires field from portal's related TO):
```xml
<SortList>
  <Sort type="Ascending">
    <Name>TableOccurrenceName::FieldName</Name>
  </Sort>
</SortList>
```
Sort is silently dropped if the field does not belong to the portal's relationship context. ✓

With sort:
```xml
<SortList>
  <Sort type="Ascending">
    <Name>TO::FieldName</Name>
  </Sort>
</SortList>
```

### §10.4 FilterCalc

Omit entirely when no filter. ◎

Element order in `PortalObj`: `TableAliasKey` → `SortList` → `FilterCalc` → `Styles` → field `Object` elements. ◎

### §10.5 Portal field FieldObj flags

| Value | Meaning |
|---|---|
| `32` | Enterable ◎ |
| `36` | Not enterable ◎ |

---

## §11 TabControl / TabPanel

```xml
<Object type="TabControl" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="50" left="10" bottom="400" right="600"/>
  <TabControlObj tabHeight="20" visPanelKey="2" defaultVisPanelKey="2"
                 visPanelIndex="0" defaultVisPanelIndex="0"
                 tabWidthModifier="70" tabJustification="1" tabFlagSet="312">
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

**TabPanel content is NOT nested inside TabPanel elements.** Content objects are placed as layout siblings at absolute coordinates overlapping the TabControl bounds. ◎

### §11.1 TabControlObj attributes

| Attribute | Notes |
|---|---|
| `tabHeight` | Height of the tab strip in points ◎ |
| `visPanelKey` | Key of currently visible panel ◎ |
| `defaultVisPanelKey` | Key of panel shown by default ◎ |
| `visPanelIndex` | 0-based index of visible panel ◎ |
| `defaultVisPanelIndex` | 0-based index of default panel ◎ |
| `tabWidthModifier` | Tab label width adjustment ◎ |
| `tabJustification` | `0` = left, `1` = centre, `2` = right ◎ |
| `tabFlagSet` | Observed values: `264`, `312`, `328`. Use `312` ◎ |

TitleCalc accepts a bare FM expression or a quoted string literal:
```xml
<!-- Static -->
<TitleCalc><Calculation><![CDATA["My Tab"]]></Calculation></TitleCalc>
<!-- Dynamic -->
<TitleCalc><Calculation><![CDATA[Let(n = TO::count; "Tab" & Case(n > 0; " (" & n & ")"))]]></Calculation></TitleCalc>
```

---

## §12 SlideControl / SlidePanel

```xml
<Object type="SlideControl" key="1" LabelKey="0" flags="0" rotation="0">
  <Bounds top="50" left="4" bottom="400" right="762"/>
  <SlideControlObj visPanelKey="1047" visPanelIndex="4"
                   dotSize="9" slideFlagSet="1">
    <Object type="SlidePanel" key="2" LabelKey="0" flags="0" rotation="0">
      <Bounds top="0" left="0" bottom="350" right="758"/>
      <SlidePanelObj slidePanelFlagSet="0"/>
      <Styles>
        <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
      </Styles>
    </Object>
  </SlideControlObj>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
</Object>
```

SlidePanel `Bounds` are relative to SlideControl. Content objects are layout siblings, not nested — same pattern as TabControl. ◎  
No `TitleCalc` — navigation is via dot indicators. ◎

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

`numOfObjs` = count of grouped child objects. Child objects follow `Styles` inside `GroupButtonObj`. ◎

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
  <TextObj flags="0"/>
  <PopoverButtonObj>
    <Object type="Popover" key="2" LabelKey="0" flags="0" rotation="0">
      <Bounds top="50" left="10" bottom="200" right="300"/>
      <PopoverObj/>
      <TitleCalc>
        <Calculation><![CDATA["My Popover"]]></Calculation>
      </TitleCalc>
      <Styles>
        <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
      </Styles>
    </Object>
  </PopoverButtonObj>
  <HideCondition findMode="False">
    <Calculation><![CDATA[IsEmpty($$var)]]></Calculation>
  </HideCondition>
  <LabelCalc>
    <Calculation><![CDATA["Open"]]></Calculation>
  </LabelCalc>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
</Object>
```

Popover `Bounds` are absolute layout coordinates. ◎

Element order in `Popover`: `Bounds` → `Styles` → `TitleCalc` → `PopoverObj`. ✓

---

## §15 WebViewer (ExternalObject)

```xml
<Object type="ExternalObject" key="1" LabelKey="0" name="wv1"
        flags="0" rotation="0">
  <Bounds top="10" left="10" bottom="200" right="500"/>
  <ExternalObj typeID="WEBV" typeIndex="0" externalFlagSet="32865">
    <ExtendedAttributes fontHeight="10" graphicFormat="0">
      <NumFormat flags="0" charStyle="0" negativeStyle="0" currencySymbol=""
                 thousandsSep="0" decimalPoint="0" negativeColor="#0"
                 decimalDigits="0" trueString="" falseString="No"/>
    </ExtendedAttributes>
    <Styles>
      <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
    </Styles>
    <Calculation index="0"><![CDATA["https://example.com"]]></Calculation>
  </ExternalObj>
</Object>
```

- `externalFlagSet="32865"` is the standard value ◎
- `name` attribute on Object element when targeted by `Perform JavaScript in Web Viewer` ◎
- URL or full HTML string in `Calculation index="0"` ◎
- Chart (`typeID="CHRT"`) not generatable

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

Multiple `<Item>` elements stacking in one block is plausible but not yet verified (observed fields had a single item). ◎

The `Item`, `Condition`, `op`, `Calculation` and `RangeBegin` structure round-trips; `op="3"` with a populated `RangeBegin` is verified. The `Item flags` meanings (§16.1) and the non-`3` `op` values (§16.2) are inherited observations (◎), not yet isolated by round-trip.

### §16.1 Item flags (inherited observation)

| Bit | Value | Meaning |
|---|---|---|
| 0 | 1 | Change fill/background colour ◎ |
| 1 | 2 | Change text/foreground colour ◎ |
| 2 | 4 | Change icon colour ◎ |
| 7 | 128 | Icon-only format ◎ |

Common values: `5` (fill+icon), `7` (fill+text+icon), `129` (icon only). ◎

### §16.2 Condition op

| Value | Meaning |
|---|---|
| `3` | Equal to ✓ (captured) |
| `0` | Formula is ◎ |
| `5` | Greater than ◎ |
| `6` | Less than ◎ |

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
`<TriggerText>` carried the script name; in the captures, where the script was named `beee`, `TriggerText` came back as `"beee"` (quote-wrapped) while the `<Script name>` attribute was unquoted. Reproduce `TriggerText` as observed; the reason for the wrapping was not established. ◎
Multiple `<Trigger>` elements stack inside one `<ScriptTriggers>` block, in event-id order. ✓

---

## §19 Button labels, LabelCalc, and icon streams

### §19.1 Standalone button labels are static text

A standalone `Button` object's label is literal text carried as `<Data>` inside the `TextObj`'s `CharacterStyleVector` AND `ParagraphStyleVector`, with a matching `CharacterStyle`. A standalone button does NOT use `LabelCalc`. ✓

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

`LabelCalc` drives a dynamic label on a **button-bar segment**, not on a standalone button. It is a child of the segment `Object`, positioned **after `ButtonObj`**. When present, the segment's `TextObj` `<Data>` is empty. ✓

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
| `.text` | Field, Text | Text/paragraph styling within the object ◎ |
| `.icon` | Button, TabPanel | Icon colour and styling ◎ |
| `.row` | Portal | Default row background ◎ |
| `.row_alt` | Portal | Alternating row background ◎ |
| `.row_active` | Portal | Active/selected row background ◎ |
| `.button_bar_divider` | ButtonBar | Divider line between segments ◎ |
| `.contents` | Portal, TabPanel | Inner content area ◎ |
| `.inner_border` | Various | Inner border styling ◎ |
| `.repeat_border` | Field | Repeating field border ◎ |
| `.baseline` | Text | Bottom border only (underline style) ◎ |

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

**Before `Bounds`** (each corroborated individually; the relative order of these two when both are present was not captured):
- `ScriptTriggers` *(if present)* ✓
- `ConditionalFormatting` *(if present)* — a direct `Object` child, before `Bounds`. ✓ (Do NOT place it inside `FieldObj`; a generated object with CF inside `FieldObj` had the CF dropped on paste.)

**Then:**
1. `Bounds`
2. `ToolTip` *(if present)* — after `Bounds`, before the typed inner element ✓
3. Typed inner element (`FieldObj`, `TextObj`, `ButtonObj`, `RectObj`, etc.). For a `FieldObj` the internal order is: `Name`, `ExtendedAttributes`, `Styles`, `PlaceholderText` *(if present)*, `DDRInfo` (the latter carries any `ValueList` descriptor). ✓
4. `RenderFormat` *(shapes only)* ✓
5. `CanEntryCalc` *(if present, fields only — FileMaker 2026)* ✓
6. `HideCondition` *(if present)* ✓

**Trailing children.** `CanEntryCalc` and `HideCondition` are the LAST children of the `Object`, after the entire typed inner element block (and after `RenderFormat` on shapes). When both are present, FileMaker normalises them to **`CanEntryCalc` then `HideCondition`**, regardless of paste order. ✓

**Scope.** `CanEntryCalc` is for fields only — on a non-field object it can cause the whole object to fail to paste (§27), so never attach it elsewhere. `HideCondition` applies to any object type, including shapes, where it sits after `RenderFormat`. ✓

**Object flags set by behavioural elements** (do not set these when generating; FileMaker sets them from object state):
- bit 0 (1) — has `ConditionalFormatting` ✓
- bit 2 (4) — has `HideCondition` ✓
- bit 14 (16384) — has `ToolTip` ✓ (isolated from a tooltip-only-among-before-Bounds field). Whether `PlaceholderText` alone also sets this bit was not isolated; `PlaceholderText` definitely sets `FieldObj` bit 17 (131072), which is a separate flag.

For container types (ButtonBar, TabControl, Portal, PopoverButton, SlideControl, Group), nested child `Object` elements are inside the typed inner element, after that element's `Styles`.

---

## §22 Step reference


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

---

## §23 Silent failure modes

- `type="FMObjectList"` instead of `"LayoutObjectList"` — entire paste dropped silently ✓
- Tabs instead of spaces — elements dropped silently ✓
- Missing required Step children — step parameters dropped silently ✓
- Unknown `ThemeName` — FM substitutes the file's default theme ◎

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
externalFlagSet:  "32865" (WebViewer)
```

Do not generate Object flags bits 14, 16, 24, 28, 30, 31 — FM sets these from object state.

---

## §25 Object styling and serialization model

This is the core of how a styled object is written out. It is **theme-independent** — the structure below is identical regardless of which theme is active. Only the values differ between themes (see §27).

### §25.1 The three CSS elements

Inside `<Styles>`, an object carries up to three CSS blocks plus the theme name:

- **`<LocalCSS>`** — the *changed-property delta only*, grouped by state. Present when the object overrides its theme defaults.
- **`<CustomStyles><Name>…</Name></CustomStyles>** — a reference to a named theme style. The `Name` is the style's internal `FM-`UUID (e.g. `FM-11711CFC-75AA-486A-B945-C847FEF44E34`), NOT its display name. A style shown as "fred" in the theme is stored as a UUID; the display name is only a label. The reference is a pointer: the style's actual appearance lives in the theme keyed by that id, not in the `CustomStyles` block. ✓
- **`<FullCSS>`** — the *full computed merge*: every property, the complete resolved appearance. FileMaker ALWAYS recomputes this from the destination theme on paste.
- **`<ThemeName>`** — closes the `Styles` block.

### §25.2 The four cases

| Case | Elements emitted, in order |
|---|---|
| Theme default (no override) | `FullCSS`, `ThemeName` |
| Local override | `LocalCSS`, `FullCSS`, `ThemeName` |
| Named style applied | `CustomStyles`, `FullCSS`, `ThemeName` |
| Named style + local override | `LocalCSS`, `CustomStyles`, `FullCSS`, `ThemeName` |

The two `CustomStyles` cases are generatable: emit the style's `FM-`UUID in `<CustomStyles><Name>` and it binds on paste (see §25.3). The id must be a real style id from the target theme, not a display name.

### §25.3 Generation rule

When generating, emit a minimal `FullCSS` (the handful of properties you care about) plus `ThemeName`. **FileMaker recomputes `FullCSS` from the destination theme's tokens on paste**, expanding a minimal block to the full baseline. For deliberate overrides, also emit a `LocalCSS` delta containing only the changed properties grouped under the relevant `self:STATE .selector` heading. Do not hand-write the full baseline — let FileMaker compute it.

**Applying a named style by generation works, by id. ✓** Emit `<CustomStyles><Name>FM-...UUID...</Name></CustomStyles>` carrying the style's real internal id, plus a minimal `FullCSS`, and on paste the object binds to that style and renders it. The binding survives a subsequent copy (the `CustomStyles` id comes back intact). A display name does NOT work — `<Name>fred</Name>` has nothing to bind to and the object falls back to its base appearance; the id is everything.

The constraint is sourcing the id. A style's `FM-`UUID lives in the theme, not in any catalogue the clipboard format exposes. You can only obtain it from an object that already carries the style: one the user supplies as an exemplar, or one present in a copied set. There is no way to derive a style's id from its display name through this format. To give an object a named style's *look* without its id, reproduce the appearance as a `LocalCSS` override instead — visually equivalent, but not linked to the style, so it will not track later edits to it.

### §25.4 State vocabulary

`normal`, `hover`, `pressed`, `focus`, `checked`, `checkedfocus`, `placeholder`, `droptarget`. Each appears as a `self:STATE .selector { … }` block. Which states a theme actually populates is theme-dependent — a minimal theme may emit only `normal`, `focus`, and `placeholder` for a field, while a richer theme adds `hover` and `droptarget`. The vocabulary is universal; the populated subset is not. ✓

### §25.5 Fill and gradient

- Solid fill emits `background-color` together with `background-image: none` and `border-image-source: none`. ✓
- Gradient fill emits `-webkit-gradient(...)` in `background-image` with a transparent `background-color`. Radial, linear, and multi-stop (`color-stop(0.5,...)`) all round-trip. ✓

### §25.6 Retheming objects to a named style

Because a named style applies by id (§25.3), an object's style can be reassigned by rewriting its `<CustomStyles><Name>` to a different style's id, and an overridden object can be put onto a style by removing its `<LocalCSS>` and inserting a `<CustomStyles>` id. Verified end to end: a set of fields each carrying a `font-size` `LocalCSS` override, restyled by stripping the override and adding a style id, pasted clean and on-style. ✓

Method:
1. Identify the target objects. "Overridden" objects are those with a `<LocalCSS>` block (the local-override and named-style-plus-override cases of §25.2).
2. For each, drop the `<LocalCSS>` block and replace the `<Styles>` contents with `<CustomStyles><Name>STYLE-ID</Name></CustomStyles>`, a minimal `<FullCSS>`, and the `<ThemeName>`.
3. The `STYLE-ID` must be sourced from an object that already carries the style (an exemplar, or another object in the copied set) — see §25.3.

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
| 2^7 | 128 | double-underline |
| 2^8 | 256 | bold |
| 2^9 | 512 | italic |
| 2^10 | 1024 | underline (single) |
| 2^12 | 4096 | highlight |
| 2^13 | 8192 | condensed |
| 2^14 | 16384 | expanded |

Notes:
- **Case transform is a two-bit field.** uppercase = 16, lowercase = 32, capitalize = both (48), none = 0. ✓
- **Single and double underline are independent bits** (1024 and 128), not two values of one field. ✓
- **Bold and italic are coordinated three-part changes.** Each sets its Face bit AND the CSS (`font-weight: bold` / `font-style: italic`) AND swaps the postscript font-family variant (e.g. `HelveticaNeue-Bold`, `HelveticaNeue-Italic`). Generate all three together. ✓
- Underline and strikethrough also travel as CSS (`-fm-underline: underline` | `double-underline`, `-fm-strikethrough: true`) alongside their Face bits. ✓
- Bits 2^6 (64), 2^11 (2048), 2^15 (32768) are unobserved — meaning unknown. Do not set them.

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

**Frontier.** Only the field-entry state (`CanEntryCalc`) is captured. Other 2026 calc-driven access states, if any, will have their own trailing elements with their own tag names — not yet observed, do not guess them.

---

## §28 Theme independence (proven across two themes)

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

Do not assert these as fact; mark them ◎/○ if generating near them:

- Relative order of `ScriptTriggers` and `ConditionalFormatting` when both precede `Bounds` (each is before `Bounds`; their order is not pinned).
- Conditional format payload round-trip and the `Item flags` / `op` meanings beyond `op="3"` (§16.1/§16.2).
- 2026 calc-driven access states other than `CanEntryCalc` — tag names not observed.
- `Face` bits 2^6 (64), 2^11 (2048), 2^15 (32768) — unobserved; do not set (§26).
