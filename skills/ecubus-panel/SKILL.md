---
name: ecubus-panel
description: >-
  Design, edit and debug EcuBus-Pro Panels (free-layout Panel editor/runtime):
  controls on the canvas, groups/tabs, binding controls to DBC/LDF signals or
  user/system variables, HTML controls with the window.panel bridge, button
  actions, Remember value / initial values, and legacy (form-create) panel
  migration. Use when the user asks to build or fix a Panel, edit
  data.panels in an .ecb project, wire a panel to signals/variables, write HTML
  control markup/script, or reproduce panel_showcase / panel_html examples.
---

# EcuBus Panel

Help users build and debug **Panels** in EcuBus-Pro: free-layout UIs (editor +
runtime view) stored inside the `.ecb` project, whose controls read and write
CAN/LIN signals and variables while measurement runs.

This skill is about **authoring Panel configs** (in the UI or directly in the
`.ecb` JSON) and the scripts that feed them, not about changing EcuBus-Pro source.

## Panel vs. other windows

| Feature | What it is | Stored in `.ecb` |
|---------|-----------|------------------|
| **Panel** | Free-layout control surface (buttons, sliders, LEDs, gauges, HTML…) bound to signals/variables | `data.panels` |
| Interaction (IA) | Per-node CAN/LIN/SOME-IP frame send tables (manual/periodic) | `data.ia` |
| Graph / Gauge / Data | Plots of signals/variables over time | `data.graphs`, `data.guages`, `data.datas` |
| Network / nodes | Script nodes (`ecu.ts`, test nodes) attached to devices | `data.nodes` |

A Panel does **not** send frames by itself. Signal writes only update the
transmit database values; a periodic frame (IA window or node script) must
actually transmit them.

## Prerequisites (do not invent)

Before designing a panel, confirm from the project / user:

1. **Databases**: `data.database.can` / `data.database.lin` entries — the map key
   (`dbKey`), `name` (`dbName`), frame `id`, signal names.
2. **Variables**: `data.vars` entries — `id`, `name`, `type` (`user` | `system`),
   `value.type` (`number` | `string` | `array`), `parentId` (namespace path),
   `min`/`max`/`unit`/`enum` for numbers.
3. **Who transmits / reacts**: which IA window or node script sends the frames
   that carry written signals, and which script reacts to written variables.
4. **Intent per control**: display-only vs. write, raw vs. physical value,
   momentary vs. toggle button, etc.

If a signal or variable does not exist yet, ask the user to add it (or add it
to `data.vars` explicitly). Never bind to names that are not in the project.

## Workflow

```
Task Progress:
- [ ] 1. Read the project: data.database (can/lin), data.vars, data.panels, nodes/IA
- [ ] 2. Decide controls per signal/variable (see Choosing a control)
- [ ] 3. Layout: canvas size, groups/tabs (parentId/tabId), positions on 8 px grid
- [ ] 4. Bind each control; copy binding shapes exactly (patterns.md)
- [ ] 5. Set behavior: readOnly, pressValue/releaseValue, toggle, rememberValue, initialValue
- [ ] 6. HTML controls: write htmlContent + scriptContent against window.panel only
- [ ] 7. If editing JSON directly: validate ids, parentId/tabId, binding fields
- [ ] 8. Tell the user to reopen the project, start measurement, and what to check
```

Prefer the editor UI (drag from palette / Data tab, property inspector) when the
user is in the app; it fills binding fields from the real database. Edit `.ecb`
JSON directly only when asked, and then **close the project first**, edit, and
reopen — the app keeps the project in memory and overwrites the file on save.

## Where Panels live

- Project file: `data.panels: Record<panelId, PanelItem>`
- `PanelItem = { id, name, rule: any[], options: Object, document?: PanelDocument }`
  - `document` present → new free-layout panel (schema v2).
  - `document` absent and `rule` non-empty → legacy form-create panel (see Migration).
  - New panels keep `rule: []`, `options: {}`; saving preserves existing `rule`/`options`.
- Panel `name` must be non-empty and unique across `data.panels` (editor rejects otherwise).
- Windows (in `project.wins`): editor `title: "panel"`, id `<panelId>`; runtime
  `title: "panelPreview"`, id `p<panelId>` with `options.params["edit-index"] = "p<panelId>"`.

```ts
interface PanelDocument {
  schemaVersion: 2
  width: number   // canvas px, clamped 160..4096 and never smaller than root controls
  height: number
  controls: PanelControl[]  // flat list; array order = z-order (later = on top)
}
```

Every `PanelControl` has the required base fields `id, type, label, x, y, width,
height, locked, fontSize, color, unit, min, max, step, initialValue, pressValue,
releaseValue, toggle, readOnly, options` plus type-specific optional fields.
Full per-type schema and defaults: [reference.md](reference.md).

## Control types (exhaustive on master)

`text`, `display`, `number`, `input`, `path`, `checkbox`, `radio`, `button`,
`startStop`, `switch`, `led`, `slider`, `select`, `progress`, `gauge`, `html`,
`image`, `group`, `tabs`.

There are no other types (no chart, table, hex-only, knob, etc.). "Hex text
editor" is the `input` control with `editorMode`; hex/binary numeric entry is
`number` with `numberFormat`.

### Choosing a control

| Need | Control | Binding |
|------|---------|---------|
| Static label/heading | `text` (label is the text) | none |
| Show a number / enum / string value | `display` | signal or any variable value type |
| Edit a number | `number` (`numberFormat` decimal/hex/binary) or `slider` | signal or number variable |
| On/off write | `switch` / `checkbox` (writes `pressValue`/`releaseValue`) | signal or number variable |
| Momentary or toggle write | `button` (`buttonAction` `write`, `toggle`) | signal or number variable |
| Pick one of N | `select` / `radio` (`options`) | signal or number variable |
| Status lamp | `led` (on = `pressValue`, off = `releaseValue`) | signal or number variable |
| Level indicator | `progress` / `gauge` (`min`/`max`) | signal or number variable |
| Edit text or bytes | `input` (`editorMode` text/hex/both) | **variable only**, `string` or `array` |
| Choose file/dir/save path | `path` (`pathMode`) | **variable only**, `string` |
| Start/stop measurement | `startStop`, or `button` with `buttonAction` start/stop | none |
| Open a file/folder or another panel | `button` with `buttonAction` `openFile` / `openPanel` | none |
| Logo / picture | `image` (`imageSrc` data URL) | none |
| Custom UI / logic | `html` (`htmlContent`, `scriptContent`) | none (uses `window.panel`) |
| Containers | `group`, `tabs` | none |

Binding acceptance is enforced by `acceptsBinding` (`free/sources.ts`); the
editor refuses drops that violate it and the runtime treats them as unwritable.

## Bindings

`control.binding` is one of:

```ts
{ kind: 'signal',   node: GraphNode<GraphBindSignalValue> }
{ kind: 'variable', node: GraphNode<GraphBindVariableValue> }
```

The same `GraphNode` shape used by Graph windows. Field names that matter:

- **Signal** `node.id` = `can.<dbName>.<frameName>.signals.<signalName>` (or
  `lin.…`) — this is the **live-update subscription key**.
  `bindValue = { dbKey, dbName, frameId, signalName, startBit, bitLength }`.
  `node.yAxis.enums` (`[{label, value}]`) seeds `options` on first bind.
- **Variable** `node.id` = variable id (also the live-update key).
  `bindValue = { variableId, variableType, variableName, variableFullName, variableValueType, stringRange? }`.
  `variableFullName` is the dot-joined `parentId` chain of names
  (e.g. `Group.Sub.Level`). `node.yAxis = { min, max, unit }` for number vars;
  `stringRange` mirrors the variable `enum` (`[{name, value}]`).

First bind (`applyBinding`) copies `min`/`max`/`unit`/`options` from the source;
rebinding keeps the control's existing range/options.

A binding is **valid** at runtime only if (`FreePanelView.vue`):
- variable: `vars[variableId]` exists with a `value`, `type === variableType`,
  full path equals `variableFullName`, and `value.type === variableValueType`;
- signal: `database.can|lin[dbKey].name === dbName` and the frame/signal exist
  (`frameId` + `signalName` for CAN; `signalName` for LIN).

Invalid bindings show an error and the control is disabled. Renaming a variable,
moving it to another parent, or re-importing a DBC under a new key breaks them.

### Writes

- Only while measurement is running, never for `readOnly` controls.
- Variables: only `user` variables of type number/string/array are writable
  (`ipc-var-set` with `variableFullName`). System variables are display-only.
- Signals: written via `ipc-signal-set` with name `<dbName>.<signalName>`.
  A **number is a raw value**; a **string is a physical value / enum label**.
  `number` controls with `numberValueType: 'physical'` send `String(value)`.
- Signal writes are refused (control shows "ambiguous") when several CAN/LIN
  databases share `dbName`, or the signal name appears in more than one frame of
  that CAN database (the write path addresses signals by `db.signal` only).
- Physical write on `number` needs a usable factor/offset (CAN) or a
  `physicalValue` encoding (LIN); otherwise it shows "physical unavailable".
- `radio` only writes values present in `options`; `input` with an `array`
  variable only writes integer bytes 0..255.

### Reads

At measurement start the runtime subscribes to `window.logBus` by
`binding.node.id`. `display`/`number` bound to a signal show the physical value
when `numberValueType === 'physical'`, otherwise the raw value. All other
controls use raw values. Several controls bound to the same source update together.
When stopped, only controls bound to user variables show a value (the start /
stored value); signal- and system-variable-bound controls show `—`.

## Runtime and measurement lifecycle

- **Start**: `panelStartValues` computes start values for writable controls bound
  to user variables with `rememberValue === false` and writes them into the
  dataset sent to `ipc-global-start` (number vars ← `initialValue`; string vars
  ← `initialText`; array vars ← `initialText` parsed as hex/text per `editorMode`).
  If several controls write the same variable and any has `rememberValue: false`,
  that control's initial value wins.
- Node scripts' `Util.Init` runs after and can override these values (the
  showcase `simulation.ts` re-applies `getVar(...)` explicitly).
- **Stop**: for writable controls with `rememberValue !== false` (default on),
  the bound user variables' last values are copied back into `data.vars[*].value.value`
  (`rememberedVariableIds` + `ipc-var-values`). The user must save the project to
  persist them. System variables and signal values are never written back.
- The CLI (`ecb_cli`) starts from values saved in the `.ecb`; it does not apply
  Panel initial values.

Script side (ECB API, in node scripts): react to panel writes with
`Util.OnVar('<name>', ({ value }) => …)`, publish values with `setVar` /
`setVars`, read with `getVar`. Panel signal writes are seen by the node that
transmits that frame. See [patterns.md](patterns.md).

## Button actions

`button.buttonAction`: `write` (default; binding + press/release/toggle),
`openFile` (`actionPath`, absolute or relative to project dir; only folders and
`.pdf .txt .md .csv .png .jpg .jpeg .gif .bmp .webp`), `openPanel`
(`actionPanelId` = another panel id, opens its runtime window), `start`, `stop`.
Non-`write` actions clear the binding. The editor only offers `start`/`stop` if
already set — prefer the `startStop` control for measurement control.

## HTML control

`type: 'html'` renders `htmlContent` + `scriptContent` in a sandboxed iframe
(`sandbox="allow-scripts"`, CSP: no network, no external scripts/styles, images
only `data:`/`blob:`). `htmlContent` (markup, `<style>`, inline `<script>`) goes
into `<body>`; `scriptContent` runs after it and is **not executed in the editor**.
No Node, Electron, `require`, `fetch` or `import 'ECB'` — only `window.panel`:

| Method | Returns | Notes |
|--------|---------|-------|
| `getVar(name)` | value | name = full path or bare name; system vars allowed |
| `setVar(name, value)` | `true` | running only; value type must match var type |
| `getSignal('db.signal')` | `{name, value, rawValue, physicalValue}` | latest sample, else DB value |
| `setSignal('db.signal', v)` | `true` | running only; number = raw, string = physical/enum label |
| `onVar(name, cb)` | `Promise<unsubscribe>` | cb receives the value |
| `onSignal('db.signal', cb)` | `Promise<unsubscribe>` | cb receives `{rawValue, physicalValue, …}` |
| `isRunning()` | boolean | |

All methods are async (postMessage RPC). Ambiguous signal names reject.
`setSignal` success does not mean the frame was transmitted.

For maintainable HTML, keep sources in sibling files and copy them into the
`.ecb` with a sync script like `resources/examples/panel_html/sync-panel.mjs`
(only the content inside `.ecb` is used at runtime). Recipe: [patterns.md](patterns.md).

## Compatibility / migration (do not break)

- Legacy panels (`rule` = form-create rules, no `document`) are converted on
  project open by `upgradeLegacyPanels` (`stores/panelCompatibility.ts`) **only**
  if every rule is a supported simple type (`TText`, `inputNumber`, `slider`,
  `BButton`, `select`) with no unsupported options/behavior and compatible
  bindings (variable bindings need a value type; non-`display` controls need
  number variables); otherwise the panel
  is left as-is and the editor shows a migration report and can create a
  `"<name> (V2 n)"` copy. The original `rule`/`options` are retained.
- Do not delete `rule`/`options` from a `PanelItem`, and do not change
  `schemaVersion` (only `2` exists).
- Keep optional fields optional: code relies on `undefined` defaults
  (e.g. `labelPosition` → `left`, LED → `hidden`; `rememberValue !== false` = on;
  `editorMode` missing = text; `progressValuePosition` missing derives from `progressText`).
- Migrated signal bindings keep the legacy `node` verbatim; legacy ids may be
  `can.<db>.signals.<sig>` (no frame name), which differs from the live-update key.
  If a migrated display stays empty, rebind it from the Data tab.

## API source of truth (EcuBus-Pro master)

Read these instead of guessing (paths in the EcuBus-Pro repo):

- `src/preload/panel.d.ts` — `PanelControl`, `PanelDocument`, `PanelBinding`
- `src/preload/data.d.ts` — `PanelItem`, `VarItem`, `GraphNode`, `GraphBind*Value`
- `src/renderer/src/views/uds/panel/free/model.ts` — types list, defaults, containers, geometry
- `…/free/sources.ts` — source list, `acceptsBinding`, `applyBinding`
- `…/free/runtime.ts` — `canWrite`, `writeTarget`, `sendControlValue`, `PanelConnection`
- `…/free/variableStart.ts` — Remember value / start values
- `…/free/HtmlPanel.vue` — `window.panel` bridge
- `…/free/actions.ts`, `migration.ts`, `stores/panelCompatibility.ts`
- Examples: `resources/examples/panel_showcase/` (all control types, two panels,
  `simulation.ts`), `resources/examples/panel_html/` (HTML bridge, `sync-panel.mjs`)
- User docs: `docs/en/um/panel/index.md`; tests: `test/panel/*.spec.ts`

## Anti-patterns

- Do not invent control types, properties, `buttonAction`s or `window.panel` methods.
- Do not hand-write binding nodes with guessed `dbKey`/`frameId`/`node.id`; copy
  from `data.database` (CAN `node.id` includes the frame name).
- Do not bind `input`/`path` to signals, `path` to array vars, or number-only
  controls (`number`, `slider`, `switch`, …) to string/array variables.
- Do not expect writes to system variables, or writes while stopped.
- Do not send a number to `setSignal` when you mean a physical value — use a string.
- Do not assume a signal write puts a frame on the bus; ensure a periodic sender.
- Do not put controls with `parentId` pointing to a non-container, or a `tabId`
  that is not in the parent's `tabs`; child `x/y` are relative to the parent
  content area (below a 28 px header).
- Do not use external URLs, CDN scripts or `fetch` in HTML controls (CSP blocks them).
- Do not edit the `.ecb` while the project is open in EcuBus-Pro.
- Do not strip legacy `rule`/`options` or bump `schemaVersion`.
- Do not reuse control ids, tab ids or panel names.
