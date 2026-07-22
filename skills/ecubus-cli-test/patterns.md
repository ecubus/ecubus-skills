# EcuBus test patterns

Reusable helpers and idioms for ECB test scripts. Prefer copying these into the
user script (or a local util) rather than inventing new wait loops.

## Wait for CAN message

```ts
import { CanMessage } from 'ECB'

/** Wait for CAN id, or `true` for any frame. Rejects on timeout. */
const waitForCan = (id: number | true, timeout = 1000) =>
  new Promise<CanMessage>((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error(`CAN timeout id=${id}`)), timeout)
    Util.OnCanOnce(id, (msg) => {
      clearTimeout(timer)
      resolve(msg)
    })
  })
```

Usage:

```ts
test('recv id 0x1', async () => {
  const msg = await waitForCan(0x1, 3000)
  assert.equal(msg.id, 0x1)
})

test('recv any, expect 0x2', async () => {
  const msg = await waitForCan(true, 3000)
  assert.equal(msg.id, 0x2)
})
```

## Send CAN message

```ts
import { output, CanMessage, CAN_ID_TYPE } from 'ECB'

const canMsg: CanMessage = {
  id: 0x123,
  dir: 'OUT',
  data: Buffer.from([0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07, 0x08]),
  msgType: {
    idType: CAN_ID_TYPE.STANDARD,
    remote: false,
    brs: false,
    canfd: false
  }
}
await output(canMsg)
```

## Wait for UDS response

```ts
import { DiagResponse } from 'ECB'

/** Listen after binding; use exact service or `Tester.*.recv`. */
const waitForUdsRecv = (event: string, timeout = 1000) =>
  new Promise<DiagResponse>((resolve, reject) => {
    const timer = setTimeout(() => reject(new Error(`UDS timeout ${event}`)), timeout)
    Util.OnOnce(event as any, (msg) => {
      clearTimeout(timer)
      resolve(msg)
    })
  })
```

## Send UDS and assert raw response

```ts
import { describe, test, assert, DiagRequest } from 'ECB'

describe('UDS', () => {
  test('DiagnosticSessionControl default session', async () => {
    // Register wait BEFORE send
    const pending = waitForUdsRecv('Tester_can_0.*.recv', 3000)

    const req = DiagRequest.from('Tester_can_0.DiagnosticSessionControl160')
    // optional: req.diagSetParameter('diagnosticSessionType', 1)
    await req.outputDiag()

    const resp = await pending
    assert.ok(resp.diagIsPositiveResponse())
    assert.ok(resp.diagGetRaw().equals(Buffer.from([0x50, 0x01, 0x00])))
  })
})
```

Notes:

- `DiagRequest.from('<TesterName>.<ServiceName>')` — names must exist in the project.
- `diagSetParameter` / `diagSetParameterRaw` / `diagSetRaw` adjust request before send.
- `diagGetRaw()`, `diagIsPositiveResponse()`, `diagGetResponseCode()` inspect responses.
- Prefer exact `'Tester.Service.recv'` when multiple services may respond; use `*.recv` only when safe.

## beforeEach / afterEach

```ts
import { describe, test, beforeEach, afterEach, assert } from 'ECB'

const delay = (ms: number) => new Promise((r) => setTimeout(r, ms))

describe('with hooks', () => {
  beforeEach(async () => {
    // reset DUT / clear state
  })

  afterEach(async () => {
    // cleanup
  })

  test('case', async () => {
    await delay(10)
    assert.ok(true)
  })
})
```

## Skip

```ts
test.skip('not ready yet', async () => {
  assert.ok(true)
})
```

## LIN send / recv (outline)

```ts
import { output, LinMsg, LinDirection, LinChecksumType, assert } from 'ECB'

const tx: LinMsg = {
  frameId: 0x3c,
  data: Buffer.from([0x02, 0x10, 0x01, 0xff, 0xff, 0xff, 0xff, 0xff]),
  direction: LinDirection.SEND,
  checksumType: LinChecksumType.CLASSIC
}
await output(tx)

// Wait with Util.OnLinOnce(frameId, cb) + timeout, same pattern as waitForCan
```

For LIN/CAN method signatures and message types, read
`<projectDir>/node_modules/@types/ECB/index.d.ts`.
If missing, run `ecb_cli build <project.ecb> <script.ts>` once — build regenerates it.

## Util.Init in tests

In test mode, `Util.Init` still runs as script setup before enabled tests.
Use it for one-time hardware prep shared by the whole file; prefer `before` /
`beforeEach` for suite-scoped setup.

```ts
Util.Init(async () => {
  // optional global prep
})
```

## Assertion tips

- Prefer `assert.equal` / `assert.deepEqual` / `assert.ok` with a message.
- Compare buffers with `buf.equals(expected)` or `assert.deepEqual([...buf], [...expected])`.
- On timeout, let the Promise reject — that fails the test with a clear error.
