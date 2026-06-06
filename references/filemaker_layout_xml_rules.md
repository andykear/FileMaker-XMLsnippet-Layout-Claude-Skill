# FileMaker Layout XML Spec v1.0

Empirically derived from round-trip testing and analysis of 35+ production layouts across 8 real-world applications.

**v1.0 — First public release**

**✓** = round-trip verified  **◎** = observed across multiple layouts  **○** = single-observation hypothesis

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
| `rotation` | Degrees. `0` = no rotation ✓ |
| `name` | Optional. Direct layout object name (WebViewers, named ButtonBars) ◎ |

### §2.1 Object flags — generation rule

**Use `flags="0"` for all generated objects.** FM sets these from object state.

| Bit | Value | Meaning |
|---|---|---|
| 0 | 1 | Active nav state (current layout's nav button) ◎ |
| 2 | 4 | Object has a HideCondition ✓ |
| 3 | 8 | Portal field row option ◎ |
| 8 | 256 | ButtonBar segment base ◎ |
| 9 | 512 | Layout part marker (footer/trailing section) ○ |
| 14 | 16384 | Sliding/printing option ◎ |
| 16 | 65536 | Named layout object ◎ |
| 28 | 268435456 | Slide left (print layout) ◎ |
| 30 | 1073741824 | Slide up / resize part ○ |
| 31 | -2147483648 | Locked in layout mode ◎ |

Common production values (do not generate — FM sets these):

| Value | Bits | Context |
|---|---|---|
| `260` | 2,8 | Standard nav ButtonBar segment |
| `261` | 0,2,8 | Active nav ButtonBar segment |
| `65544` | 3,16 | Named ButtonBar segment |
| `65545` | 0,3,16 | Named active ButtonBar segment |
| `-2147483648` | 31 | Locked object |

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
| `1` | Drop-down list (requires `ValueList` child) ✓ |
| `2` | Pop-up menu (requires `ValueList` child) ✓ |
| `3` | Checkbox set (requires `ValueList` child) ✓ |
| `4` | Radio button set (requires `ValueList` child) ✓ |
| `5` | Unobserved — may not exist |
| `6` | Drop-down Calendar ✓ |

`displayType=6` applies to any field type that supports the control (text, number, date, time, timestamp). Container, calculation, and summary fields do not support it and remain at `displayType=0`. The calendar popup icon within the control is a separate option — see `FieldObj flags` bits 10+19. ✓

`ValueList` element (for types 1, 2, 3):
```xml
<FieldObj displayType="1" ...>
  <Name>TO::FieldName</Name>
  <ValueList>ValueListName</ValueList>
  <Styles>...</Styles>
</FieldObj>
```

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

After the typed inner element, before `LabelCalc`. Applies to any object type.

```xml
<ButtonObj .../>
<HideCondition findMode="False">
  <Calculation><![CDATA[IsEmpty($$var)]]></Calculation>
</HideCondition>
<LabelCalc>...</LabelCalc>
```

`findMode="False"` = hide in Browse only (default).  
`findMode="True"` = hide in Find mode. ◎

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
| 7 | 128 | Filter enabled ◎ |
| 8 | 256 | Allow deletion of portal records ○ |

| Scenario | Value |
|---|---|
| No sort, no filter, no scrollbar | `16` |
| No sort, no filter, scrollbar | `21` |
| Sort, no filter | `25` |
| No sort, filter | `145` |
| Sort + filter | `153` |
| Sort + filter + scrollbar | `157` |

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
    </Object>
  </TabControlObj>
  <Styles>
    <ThemeName>com.filemaker.theme.apex_blue</ThemeName>
  </Styles>
</Object>
```

Element order in `TabPanel`: `Bounds` → `Styles` → `TitleCalc`. ✓  
`TabControlObj` requires its own `Styles` block before the panel objects. ✓  
`TabPanelObj` element is a round-trip artifact — omit when generating. ✓

**TabPanel content is NOT nested inside TabPanel elements.** Content objects are placed as layout siblings at absolute coordinates overlapping the TabControl bounds. ◎

TitleCalc accepts a bare FM expression or a quoted string literal:
```xml
<!-- Static -->
<TitleCalc><Calculation><![CDATA["My Tab"]]></Calculation></TitleCalc>
<!-- Dynamic -->
<TitleCalc><Calculation><![CDATA[Let(n = TO::count; "Tab" & Case(n > 0; " (" & n & ")"))]]></Calculation></TitleCalc>
```

`tabFlagSet` values observed: `264`, `312`, `328`. ◎

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

After the typed inner element, before `Styles`. ◎

```xml
<ConditionalFormatting>
  <Item id="0" flags="7">
    <Condition op="0">
      <Calculation><![CDATA[Self = "X"]]></Calculation>
      <RangeBegin></RangeBegin>
      <RangeEnd></RangeEnd>
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
```

### §16.1 Item flags

| Bit | Value | Meaning |
|---|---|---|
| 0 | 1 | Change fill/background colour |
| 1 | 2 | Change text/foreground colour |
| 2 | 4 | Change icon colour |
| 7 | 128 | Icon-only format |

Common values: `5` (fill+icon), `7` (fill+text+icon), `129` (icon only). ◎

### §16.2 Condition op

| Value | Meaning |
|---|---|
| `0` | Formula is |
| `3` | Equal to |
| `5` | Greater than |
| `6` | Less than |

---

## §17 Object element order

Within any `Object`:

1. `Bounds`
2. Typed inner element (`FieldObj`, `TextObj`, `ButtonObj`, etc.)
3. `ConditionalFormatting` *(if present)*
4. `HideCondition` *(if present)*
5. `LabelCalc` *(if present)*
6. `Styles`
7. `DDRInfo` *(round-trip artifact — omit when generating)*

For container types (ButtonBar, TabControl, Portal, PopoverButton, SlideControl, GroupButton), nested child `Object` elements are inside the typed inner element, after `Styles`.

---

## §18 Step reference

```xml
<!-- Perform Script -->
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

## §19 Silent failure modes

- `type="FMObjectList"` instead of `"LayoutObjectList"` — entire paste dropped silently ✓
- Tabs instead of spaces — elements dropped silently ✓
- Missing required Step children — step parameters dropped silently ✓
- Unknown `ThemeName` — FM substitutes the file's default theme ◎

---

## §20 Generation defaults

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
