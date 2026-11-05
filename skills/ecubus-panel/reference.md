# Panel control reference

Field-level schema for `PanelControl` (`src/preload/panel.d.ts`) with defaults
from `createControl` (`free/model.ts`) and the properties the editor inspector
exposes per type (`FreePanelEditor.vue`). Omitted optional fields mean "default".

## Base fields (every control)

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `id` | string | uuid v4 in editor | Unique within the document; examples use readable ids |
| `type` | `PanelControlType` | — | See list below |
| `label` | string | localized type name, or bound source name | `text`: the shown text; `button`: caption |
| `labelPosition?` | `top \| bottom \| left \| right \| center \| hidden` | `left` (`led`: `hidden`) | Only for display, number, input, path, checkbox, radio, switch, led, slider, select, progress, gauge |
| `x`, `y` | number | 32, 32 | Root: canvas px. Child: relative to parent content area (below 28 px header) |
| `width`, `height` | number | per type (below) | Min 32×24; containers can't shrink below children |
| `locked` | boolean | false | Locked (or locked ancestor) blocks move/edit/rebind/delete |
| `fontSize` | number | 14 | Editor range 10–72 |
| `color` | string | `''` | Text color; progress/gauge fill color |
| `unit` | string | `''` | Copied from number variable on first bind |
| `min`, `max` | number | 0, 100 | Range for number/slider/progress/gauge/display |
| `step` | number | 1 | number/slider (> 0) |
| `initialValue` | number | 0 | Preview value; start value when `rememberValue: false` (number vars) |
| `pressValue`, `releaseValue` | number | 1, 0 | button/switch/checkbox write values; LED on/off values |
| `toggle` | boolean | false | button: toggle instead of momentary |
| `readOnly` | boolean | false | Disables writes / actions |
| `options` | `{label, value:number}[]` | `[]` (radio: `0`,`1`) | select/radio entries; seeded from enums on first bind |
| `binding?` | `PanelBinding` | — | See SKILL.md › Bindings |
| `parentId?`, `tabId?` | string | — | Container membership; `tabId` only under `tabs` |

## Default sizes (`createControl`)

| Type | W × H | Extra defaults |
|------|-------|----------------|
| text | 160 × 32 | |
| display, number, checkbox, button, select | 160 × 64 | |
| switch | 128 × 32 | |
| led | 48 × 48 | labelPosition defaults to `hidden` |
| slider, path, radio | 240 × 64 | radio options `0`/`1` |
| progress | 240 × 48 | `labelPosition: 'left'`, `progressValuePosition: 'hidden'` |
| gauge, image | 160 × 128 | |
| input | 320 × 140 | `editorMode: 'both'` |
| startStop | 220 × 48 | |
| html | 320 × 200 | `htmlContent` placeholder div, `scriptContent: ''` |
| group | 320 × 200 | |
| tabs | 320 × 200 | `tabs: [{id:'<id>-1',label:'1'},{id:'<id>-2',label:'2'}]`, `defaultTabId: '<id>-1'` |

## Type-specific fields

### `display` and `number`
| Field | Values | Default |
|-------|--------|---------|
| `numberFormat?` | `decimal \| hex \| binary` | decimal |
| `numberDecimals?` | 0–10 (decimal only) | as-is |
| `numberShowUnit?` | boolean | `true` for display, `false` for number |
| `numberShowRange?` | boolean | false |
| `numberValueType?` | `raw \| physical` (signal bindings only) | raw |
| `alarmMode?` | `none \| limits` | none |
| `alarmLower?`, `alarmUpper?` | number | — (background colored outside limits) |
| `alarmLowerColor?`, `alarmUpperColor?` | color | `#fa8072`, `#cd5c5c` |

`display` is read-only by nature and is the only non-text control that accepts
string/array variables. hex/binary require safe integers; `number` rejects
input outside `min`..`max`.

### `progress`
| Field | Values | Default |
|-------|--------|---------|
| `progressDirection?` | `right \| left \| up \| down` | right |
| `progressText?` | `value \| percent` (`hidden` legacy) | value |
| `progressValuePosition?` | `hidden \| left \| top \| right \| bottom` | derived: `hidden` if `progressText === 'hidden'`, else `top` |
| `progressDecimals?` | 0–6 | as-is |
| `progressOrigin?` | number in `min..max` | `min` (fill grows from origin) |
| `progressShowLimits?` | boolean | false (min size 80×56 when on) |

### `gauge`
Uses `min`, `max`, `unit`, `numberDecimals` (0–6, default 2 in text), `color`.

### `led`
| Field | Values | Default |
|-------|--------|---------|
| `ledShape?` | `ellipse \| rectangle \| up \| down \| left \| right` | ellipse |
| `ledOnColor?`, `ledOffColor?` | color | `#67c23a`, `#dcdfe6` |
| `ledFrame?` | boolean | true |
| `ledKeepAspect?` | boolean | true |

State: value `=== pressValue` → on, `=== releaseValue` → off, else unknown (dashed outline).

### `button`
| Field | Values | Default |
|-------|--------|---------|
| `buttonAction?` | `write \| openFile \| openPanel \| start \| stop` | write |
| `actionPath?` | string (openFile) | — |
| `actionPanelId?` | panel id (openPanel) | — |
| `buttonShape?` | `rectangle \| rounded \| circle` | rounded |
| `buttonVariant?` | `primary \| success \| warning \| danger \| info` | Element Plus default |
| `buttonPlain?` | boolean | false |

`write`: momentary sends `pressValue` on press and `releaseValue` on release;
`toggle: true` alternates between them on click.

### `switch`, `checkbox`
Write `pressValue` when on, `releaseValue` when off; checked state = value `=== pressValue`.

### `select`, `radio`
Write the selected option's numeric `value`; `radio` refuses values not in `options`.

### `slider`
`min`, `max`, `step`; writes on change (release), incoming values don't move the thumb while dragging.

### `input`
| Field | Values | Default |
|-------|--------|---------|
| `editorMode?` | `text \| hex \| both` | missing = text (new controls get `both`) |
| `initialText?` | string | `''` |

Binds only to `string` or `array` variables (writes need a `user` variable). Arrays are bytes 0..255;
`hex` text is pairs of hex digits (whitespace ignored); a string var in hex mode
must decode as UTF-8.

### `path`
| Field | Values | Default |
|-------|--------|---------|
| `pathMode?` | `file \| directory \| save` | file |
| `initialText?` | string | `''` |

Binds only to `string` variables. "…" opens the native dialog; text can be typed too.

### `startStop`
Two buttons (START / STOP) calling the global measurement start/stop. No binding.

### `image`
`imageSrc?` (data URL; editor imports PNG/JPEG/WebP/GIF ≤ 5 MB), `imageFit?`
(`contain \| cover \| fill`, default contain). Stored inline in the `.ecb`.

### `html`
`htmlContent?`, `scriptContent?` — see SKILL.md › HTML control.

### `group`
Container with a 28 px header (`label`). Children set `parentId: <groupId>`.

### `tabs`
Container with pages: `tabs: {id, label}[]`, `defaultTabId?`. Children set
`parentId: <tabsId>` **and** `tabId: <page id>`. A page with children can't be
deleted; the last page can't be deleted.

## Binding acceptance matrix (`acceptsBinding`)

| Control | Signal | Number var | String var | Array var |
|---------|:------:|:----------:|:----------:|:---------:|
| display | ✓ | ✓ | ✓ | ✓ |
| number, checkbox, radio, switch, led, slider, select, progress, gauge | ✓ | ✓ | | |
| button (`write`) | ✓ | ✓ | | |
| input | | | ✓ | ✓ |
| path | | | ✓ | |
| button (other actions), startStop, text, image, html, group, tabs | | | | |

Writable types (`canWrite`): number, input, path, checkbox, radio, button,
switch, slider, select — and only with a writable target (signal, or `user`
variable).

## Batch edit (multi-select)

Shared keys (`properties.ts` `batchKeys`): `unit, min, max, step, initialValue,
readOnly, pressValue, releaseValue, toggle` — offered only where every selected
control supports them. Containers, text, image, html can't be batch edited.
