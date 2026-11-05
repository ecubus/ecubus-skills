# EcuBus Panel patterns

Concrete recipes for Panel JSON, bindings, HTML controls and the node scripts
that feed them. Shapes are copied from `resources/examples/panel_showcase/` and
`resources/examples/panel_html/` on EcuBus-Pro master. Field reference:
[reference.md](reference.md).

## Panel item skeleton (`data.panels`)

```json
{
  "my-panel": {
    "id": "my-panel",
    "name": "My Panel",
    "rule": [],
    "options": {},
    "document": {
      "schemaVersion": 2,
      "width": 800,
      "height": 480,
      "controls": []
    }
  }
}
```

The map key must equal `id`. `name` must be unique among panels.

## Full base control

Every control carries all base fields. Start from this and override:

```json
{
  "id": "level", "type": "slider", "label": "Level",
  "x": 16, "y": 16, "width": 240, "height": 64,
  "locked": false, "fontSize": 14, "color": "", "unit": "%",
  "min": 0, "max": 100, "step": 1, "initialValue": 0,
  "pressValue": 1, "releaseValue": 0, "toggle": false,
  "readOnly": false, "options": []
}
```

## Signal binding from a DBC

Derive every field from `data.database.can[<dbKey>]`:

| Binding field | Source |
|---------------|--------|
| `node.id` | `` `can.${db.name}.${message.name}.signals.${signal.name}` `` |
| `node.name` | `signal.name` |
| `bindValue.dbKey` | the key of `data.database.can` |
| `bindValue.dbName` | `db.name` |
| `bindValue.frameId` | `message.id` (number, e.g. `0x100` → `256`) |
| `bindValue.signalName` | `signal.name` |
| `bindValue.startBit` / `bitLength` | `signal.start_bit` / `signal.bit_length` |
| `node.yAxis.enums` (optional) | `signal.values` → `[{ value: Number(k), label }]` |

For LIN (`data.database.lin[<dbKey>]`): `node.id` =
`` `lin.${db.name}.${frame.name}.signals.${signal.name}` ``, `frameId` = `frame.id`,
`startBit` = frame signal `offset`, `bitLength` = `db.signals[name].signalSizeBits`.

Physical display + physical write of a CAN signal (showcase `target`):

```json
{
  "id": "target", "type": "number", "label": "DBC target (%)",
  "numberValueType": "physical", "numberDecimals": 1,
  "binding": {
    "kind": "signal",
    "node": {
      "type": "signal", "enable": true, "color": "",
      "id": "can.PanelDemo.Command.signals.Target",
      "name": "Target",
      "bindValue": {
        "dbKey": "panel-demo-db", "dbName": "PanelDemo", "frameId": 257,
        "signalName": "Target", "startBit": 0, "bitLength": 16
      }
    }
  }
}
```

(Base fields omitted for brevity — include them in real JSON.)

## Variable binding

```json
"binding": {
  "kind": "variable",
  "node": {
    "type": "variable", "enable": true, "color": "",
    "id": "DemoToggle", "name": "DemoToggle",
    "bindValue": {
      "variableId": "DemoToggle",
      "variableType": "user",
      "variableName": "DemoToggle",
      "variableFullName": "DemoToggle",
      "variableValueType": "number"
    }
  }
}
```

- `variableType` / `variableValueType` must equal `data.vars[id].type` / `.value.type`.
- Nested variable: `variableFullName` joins names up the `parentId` chain,
  e.g. var `Level` under parent `Motor` → `"Motor.Level"`; `variableName` stays `"Level"`.
- Optional `node.yAxis: { min, max, unit }` and `bindValue.stringRange`
  (`[{name, value}]` from `value.enum`) only seed the control on first bind in
  the editor; when writing JSON, set `min`/`max`/`unit`/`options` on the control yourself.

## Common control recipes

| Goal | Key fields |
|------|-----------|
| Momentary button → var 1 while held | `type: "button"`, `toggle: false`, `pressValue: 1`, `releaseValue: 0`, variable binding |
| Latch button | same + `toggle: true` |
| LED from a status var | `type: "led"`, `pressValue: 1` (on), `releaseValue: 0` (off), `ledOnColor: "#ef4444"` |
| Enum select from DBC | `type: "select"`, signal binding, `options: [{label, value}]` from `signal.values` |
| Read-only number | `type: "display"` (or `number` + `readOnly: true`) |
| Hex byte editor | `type: "input"`, `editorMode: "hex"`, array variable |
| Open another panel | `type: "button"`, `buttonAction: "openPanel"`, `actionPanelId: "<panel id>"`, no binding |
| Open a project doc | `type: "button"`, `buttonAction: "openFile"`, `actionPath: "DEMO.md"` (relative to project dir) |
| Start/Stop | `type: "startStop"` |
| Always start from a fixed value | writable control on a user var with `rememberValue: false` + `initialValue` (or `initialText`) |

## Containers

Group child (coordinates are relative to the group's content area):

```json
{ "id": "commands", "type": "group", "label": "Commands", "x": 16, "y": 16, "width": 350, "height": 450, "...": "base fields" }
{ "id": "auto", "type": "switch", "label": "Automatic sweep", "parentId": "commands", "x": 14, "y": 16, "width": 320, "height": 44, "...": "base fields" }
```

Tabs: the container lists pages; children name both parent and page.
Containers can be nested (showcase: group `commands` sits inside tabs `pages`,
so it has `parentId: "pages", tabId: "overview"`, and its own children use
`parentId: "commands"`; `tabId` is only used when the direct parent is a `tabs`).

```json
{ "id": "pages", "type": "tabs", "label": "Demo", "x": 20, "y": 72, "width": 1160, "height": 650,
  "tabs": [{ "id": "overview", "label": "Overview" }, { "id": "inputs", "label": "Inputs" }],
  "defaultTabId": "overview", "...": "base fields" }
{ "id": "speed", "type": "display", "parentId": "pages", "tabId": "overview", "...": "..." }
```

Keep each child inside its parent: `x + width <= parent.width` and
`y + height <= parent.height - 28`.

## HTML control

Script field pattern (from the showcase): read once, subscribe, write with error display.

```js
const $ = (selector) => document.querySelector(selector)
let fill = 0
const showFill = (value) => {
  fill = Number(value) || 0
  $('#fill').textContent = fill + ' %'
}
const setFill = (value) =>
  panel
    .setVar('DemoFill', Math.max(0, Math.min(100, value)))
    .then(() => ($('#msg').textContent = ''))
    .catch((error) => ($('#msg').textContent = error.message))
panel.getVar('DemoFill').then(showFill)
panel.onVar('DemoFill', showFill)
panel.onSignal('PanelDemo.Speed', (signal) => {
  $('#speed').textContent = Number(signal.physicalValue).toFixed(1)
})
```

Rules for HTML scripts:

- Everything is async; wrap top-level `await` in an async function.
- Keep the unsubscribe functions returned by `onVar`/`onSignal`; release them on `pagehide`.
- Poll `panel.isRunning()` (e.g. every 500 ms, as `panel_html/control.js` does) to
  enable/disable inputs; writes reject with `Panel is stopped` otherwise.
- `setSignal('Db.Sig', '45')` = physical 45; `setSignal('Db.Sig', 450)` = raw 450.
- Styles inline in `htmlContent`; images only as `data:`/`blob:` URLs.

Keeping HTML in source files (adapt `panel_html/sync-panel.mjs`; run with the
project closed, then reopen):

```js
import { readFileSync, writeFileSync } from 'node:fs'

const file = new URL('./MyProject.ecb', import.meta.url)
const text = readFileSync(file, 'utf8')
const eol = text.includes('\r\n') ? '\r\n' : '\n'
const project = JSON.parse(text)
const control = project.data.panels['<panelId>'].document.controls.find((c) => c.id === '<htmlControlId>')
const source = (name) => readFileSync(new URL(name, import.meta.url), 'utf8').replace(/\r\n/g, '\n')
control.htmlContent = source('./control.html')
control.scriptContent = source('./control.js')
writeFileSync(file, JSON.stringify(project, null, 2).replace(/\n/g, eol) + eol)
```

## Node script hooks that feed a panel

Panel controls write variables/signals; a node script (ECB API) reacts and
publishes values back (from `panel_showcase/simulation.ts`):

```ts
import { getVar, setVar, setVars } from 'ECB'

Util.OnVar('DemoPulse', ({ value }) => {
  if (value === 1) setVar('DemoCounter', Number(getVar('DemoCounter')) + 1)
})

Util.Init(() => {
  // Re-apply current values so the script does not override panel start values
  setVars({ DemoLevel: getVar('DemoLevel'), DemoCounter: 0 })
})
```

Signals written from the panel only change the transmit database; transmit
them with a periodic IA entry or `output(...)` in a node script, and read
received frames with `Util.OnCan(id, …)`.

## From DBC + variables to a working panel

1. List DBC frames/signals and user variables; mark each as read or write.
2. Received signals → `display` (`numberValueType: "physical"` for scaled values), `gauge`, `progress`, `led`.
3. Transmitted signals → `number`/`slider`/`select`/`switch`; confirm a periodic sender exists for the frame.
4. Operator parameters → user variables + writable controls; decide `rememberValue`.
5. Add a `startStop` control and group related controls; use `tabs` for >1 screen.
6. Size the canvas to fit root controls; snap positions to the 8 px grid.
7. Reopen the project, open the panel runtime, start measurement, check the
   status bar `Issues` count is absent and each control updates/writes.

## Debugging checklist

| Symptom | Likely cause |
|---------|-------------|
| Control shows "invalid binding", disabled | variable renamed/moved/retyped, or `dbKey`/`dbName`/`frameId`/`signalName` not in database |
| "ambiguous" | duplicate `dbName` across databases, or signal name in several frames of one CAN DB |
| "physical unavailable" | `number` with `numberValueType: physical` on a signal without usable factor (CAN) / physical encoding (LIN) |
| Display stays `—` while running | `node.id` doesn't match the live key (`can.<db>.<frame>.signals.<sig>` / variable id), or no frames received |
| Write control always disabled | measurement stopped, `readOnly`, system variable, or binding not accepted for this type |
| Signal write has no effect on bus | no periodic transmission of that frame |
| Value resets every start | `rememberValue: false`, or node script `Util.Init` sets it |
| Value not kept after stop | project not saved after measurement stopped |
| HTML script does nothing in editor | `scriptContent` is not executed in the editor; use runtime view |
| HTML fetch/CDN fails | CSP `connect-src 'none'`, no external sources |
