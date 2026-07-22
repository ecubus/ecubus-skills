---
name: ecubus-cli-test
description: >-
  Write EcuBus-Pro CLI/HIL test scripts (TypeScript) using the ECB test framework
  (describe/test/assert, Util, DiagRequest, output). Use when the user asks to
  write or modify EcuBus test cases, ecb_cli test scripts, HIL tests, CAN/LIN/UDS
  request-response tests, or when editing a node with isTest=true / test.ts in an
  .ecb project.
---

# EcuBus CLI Test Scripts

Write TypeScript test scripts for EcuBus-Pro after the user has configured the
`.ecb` project (devices, testers/services, test node + script entry).

## Prerequisites (do not invent)

Before writing tests, confirm from the project (`.ecb` / UI config / user):

1. **Test node**: `isTest: true`, `name` (CLI `<name>`), `script` path (e.g. `test.ts`)
2. **Channels**: node bound to the right CAN/LIN/ETH devices
3. **Testers / services**: exact names for `DiagRequest.from('<Tester>.<Service>')`
4. **Expected stimulus/response**: IDs, payloads, timeouts, positive/negative UDS

If names or expected data are missing, ask. Do not invent tester/service names.

## Script rules

- Import APIs from `'ECB'` (bundled worker API). Prefer:

```ts
import {
  describe,
  test,
  assert,
  beforeEach,
  afterEach,
  before,
  after,
  DiagRequest,
  DiagResponse,
  CanMessage,
  output
} from 'ECB'
```

- `Util` is a **global** (`Util.OnCan`, `Util.OnOnce`, `Util.Init`, …). Do not import it.
- Framework is Node.js test runner wrappers: `describe` / `test` / `test.skip` / hooks.
- **Hooks (`before` / `after` / `beforeEach` / `afterEach`) MUST be inside `describe`.**
- Tests are usually `async`; fail with `assert` / `assert.equal` / `assert.deepEqual` / thrown errors.
- Always use timeouts when waiting for bus/UDS traffic; never wait forever.
- Register listeners **before** sending requests (avoid race).
- Service event names: `'<Tester>.<Service>.send'|'.recv'`; wildcards: `'<Tester>.*.recv'`.

## Workflow

```
Task Progress:
- [ ] 1. Read project: test node name, script path, testers, services, channels
- [ ] 2. Read node_modules/@types/ECB/index.d.ts (if missing: ecb_cli build once to generate it)
- [ ] 3. Map user requirements → test suites / cases
- [ ] 4. Write/update the script entry (usually test.ts); use patterns.md helpers
- [ ] 5. Build the script and fix errors until build succeeds
- [ ] 6. Tell user how to run via CLI
```

## Always build after writing

After creating or editing the test script, **always build before considering the task done**.
Do not skip this step. Build validates TypeScript/ECB API usage; fix all errors and rebuild
until it succeeds.

```bash
ecb_cli build <project.ecb> <script.ts>
```

Use the user's actual project path and script entry (from the test node's `script` field):

```bash
ecb_cli build ./my_project.ecb test.ts
```

If build fails: read the error, fix the script, rebuild. Only then suggest `ecb_cli test`.
When running tests immediately after edits, pass `-b` so the latest script is rebuilt:

```bash
ecb_cli test ./my_project.ecb <TestConfigName> -b
```
### Mapping requirements

| User need | Approach |
|-----------|----------|
| Wait for CAN frame | `Util.OnCanOnce` + timeout Promise |
| Send CAN/LIN | `output(msg)` |
| UDS request/response | `DiagRequest.from(...).outputDiag()` + `Util.OnOnce('Tester.*.recv', …)` |
| Check payload | `msg.diagGetRaw()` / `Buffer` compare, or `diagIsPositiveResponse()` |
| Setup/teardown | `beforeEach` / `afterEach` inside `describe` |
| Skip unfinished case | `test.skip('…', …)` |

### Minimal skeleton

```ts
import { describe, test, assert, DiagRequest, CanMessage } from 'ECB'

describe('Feature under test', () => {
  test('should …', async () => {
    // arrange listeners → act → assert
    assert.ok(true)
  })
})
```

Prefer the helpers in [patterns.md](patterns.md) for wait-for-CAN / wait-for-UDS.

## CLI run

```bash
ecb_cli test <project.ecb> <TestConfigName> [options]
```

`<TestConfigName>` is the test node's `name` in the user's `.ecb` project.

Options:

- `-b, --build` force rebuild script
- `-r, --report <file.html>` HTML report
- `-t, --testNamePattern <pattern>` run matching tests (regex or substring)
- `-v, --view` print test tree only
- `--log-level <error|warning|info|debug>`

```bash
ecb_cli test ./my_project.ecb "My Test Config" -b
```

Exit code non-zero when any test fails.

## API source of truth (local `.d.ts`)

Do **not** rely on online docs. Read types from the user's project:

- `<projectDir>/node_modules/@types/ECB/index.d.ts` — ECB script/test API (`describe`/`test`/`assert`, `Util`, `DiagRequest`, `output`, …) plus project-specific service/signal names
- `<projectDir>/tsconfig.json` — which scripts are included

If `index.d.ts` is missing, run build once — it **auto-generates** the types before compiling:

```bash
ecb_cli build ./my_project.ecb test.ts
```

(`compileTsc` → `createProject` / `refreshProject` writes `node_modules/@types/ECB/index.d.ts` from the `.ecb` dataset.) Then re-read the `.d.ts`. Types are written even if the script itself later fails typecheck.

For usage patterns (wait helpers, UDS send/assert), use [patterns.md](patterns.md).

## Anti-patterns

- Do not use Vitest/Jest/`node:test` imports; use ECB `describe`/`test`/`assert`.
- Do not put hooks outside `describe`.
- Do not hardcode wrong tester/service names; read them from the project.
- Do not omit timeouts on bus waits.
- Do not skip build after writing/editing the script; always `ecb_cli build` (or `test -b`) and fix errors first.
- Do not treat this as unit tests of app source; these are hardware/HIL/script tests against configured devices.
