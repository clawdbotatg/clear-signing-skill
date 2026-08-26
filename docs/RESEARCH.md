# Research notes — 2026-08-26

Two research passes behind the architecture. Condensed; both verified against live code/CI, not docs alone.

## 1. Foundry has no extension system — integrate by reading artifacts

- No plugin API. #706 (2022), #6509, #7760 all closed unimplemented into meta-issue **#8266, still open** (last activity Dec 2025). `forge lint` is config-only. No lifecycle hooks exist.
- **The pattern every real tool uses** (forge-deploy, foundry-devops, Scaffold-ETH): standalone CLI that reads `out/` + `broadcast/` after forge runs, wrapped in an npm/just task. `vm.ffi` is the only in-script escape hatch — requires `ffi = true` (security-sensitive, widely banned) and runs *before* broadcast/verify complete. Rejected.
- Don't add a custom `foundry.toml` section — forge warns on unknown sections. Use own config or flags.
- **Artifacts have everything:**
  - `broadcast/<Script>.s.sol/<chainId>/run-latest.json`: `transactions[] {transactionType: CREATE|CREATE2|CALL, contractName, contractAddress, arguments[], additionalContracts[]}`, `chain`, `receipts[]`. (Sensitive variant in `cache/` — never read that.)
  - Default `out/*.json` has no top-level devdoc/userdoc, **but** `metadata.output.{abi,devdoc,userdoc}` is always there. Cleanest zero-config read: `forge inspect <path>:<Name> abi|devdoc|userdoc --json`.
  - `extra_output = ["devdoc","userdoc"]` promotes them to top-level artifact keys if we want file-based reads.
- **Verification:** sourcify is forge's *default* verifier. `forge verify-contract` POSTs standard-JSON to Sourcify **API v2** (`v2/verify/{chainId}/{address}`, v1 died July 2026), returns a job id, `--watch` polls. **No hook, no signal file** — poll `GET /v2/contract/{chainId}/{address}` ourselves.

## 2. Tooling landscape — what to compose, what's missing

| Tool | Generate | Lint | Validate | Test | Render | Attest | State |
|---|---|---|---|---|---|---|---|
| `erc7730` py 1.0.10 (Ledger) | minimal, **emits v1**, no NatSpec | ✅ v1+v2 | ✅ | ❌ | ❌ | ❌ | active |
| Ledger web builder | UI (wraps py) | — | — | ❌ | Stax/Flex preview | ❌ | **dormant, v1-only** |
| `@ethereum-sourcify/clear-signing` 0.2.2 | ❌ | ❌ | ❌ | via runner repo | ✅ v2 | 🔶 PR #65 open (ERC-8176) | active |
| sourcify test-runner (CI-pinned @dae3cda) | ❌ | ❌ | ❌ | ✅ testsv2 | — | ❌ | active |
| Cyfrin `clearsig` 0.3.1 | from Sourcify ABI, `--v2` alpha, proxy-traversing | ❌ | ❌ | ❌ | calldata→intent | ✅ hash only | quiet since May |
| llbartekll Rust (`cs-test`, CI-pinned @10605ba) | ❌ | ❌ | ❌ | ✅ testsv2 | ✅ full v2 engine, UniFFI | ❌ | very active |
| LI.FI pipeline (lifinance/contracts) | ✅ bespoke templates | via `erc7730 format` | — | ✅ hand-rolled | ❌ | ❌ | the only automated generate→PR in the wild |
| portdeveloper/clear-signing-helper | agent playbook, address-first | via erc7730 | — | ❌ | — | ❌ | Jul 2026 |

**`erc7730 generate` detail:** fetches ABI from Sourcify/Etherscan, title-cases param names, keyword/type heuristics for formats. Zero NatSpec usage (confirmed: no devdoc/userdoc hits in source). Emits v1 input model — no `intent`, no `$schema`, no tests.

### Mergeable registry PR = (verified against workflows + merged PR #2910)

1. One entity per PR, files only under `registry/<entity>/`; names `calldata-*` / `eip712-*` / `common-*`.
2. **`testsv2/<name>.tests.json` per descriptor is mandatory** — `require-testsv2` hard-fails. Touching `common-*` or `ercs/*` requires tests for every includer.
3. Test shape: `{$schema, descriptor, dataProvider?: {tokens, addressNames, ensNames, nftCollectionNames, blockTimestamps}, tests: [{description(unique), rawTx(unsigned)|data, expected: {intent, interpolatedIntent?, owner, fields:[{label,value}]}}]}`.
4. Six CI gates: `erc7730 lint --gha` (pip **unpinned**) · `check-jsonschema` v2 · index collision check (`generate-index.js --validate`) · file names · attested descriptors are **immutable** (new version = new filename) · both reference runners (sourcify @dae3cda, rust cs-test @10605ba). Ledger device tests disabled (private deps).
5. Human review is plausibility-only (REVIEWING.md): watch for silent ABI-fetch-skip in lint, no bare-function-name intents, nothing value-bearing `visible:"never"`.
6. Known live spec hazards: `chainId` format has no fallback/name-source (#2916/#2904 — runners disagree, exact-string tests can't pass on both); `ENUM_VALUE` is one byte, `erc7730 calldata` **silently drops** descriptors with enum keys >255 (exit 0); `EnumDefinition` capped at 32 entries in the py lib but not the schema. Versioning overhaul incoming (#2897).

### The gaps we build in

1. **ABI+NatSpec → v2 draft** — nobody reads NatSpec; nobody emits v2-native (`intent`, `interpolatedIntent`, visibility judgment, `proxy`/`stateRefs`).
2. **testsv2 generation** — the biggest hole: nothing builds rawTx vectors, mines real txs, derives `dataProvider`, or renders `expected` (the runners can render — close the loop).
3. **Local CI preflight** — replicate all six gates before pushing.
4. **Generic end-to-end pipeline** — LI.FI proved it works but hand-rolled it for their own structs.

Don't rebuild: lint/format (`erc7730`), rendering (the two runners), attestation hashing (`clearsig`).
