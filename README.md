# clear-signing-skill

Deploy a contract with Foundry → get a merge-ready ERC-7730 clear-signing descriptor PR.

An agent skill + thin CLI that closes the gap between `forge script --broadcast --verify` and a green PR to the
[clear signing registry](https://github.com/ethereum/clear-signing-erc7730-registry): draft a **v2-native descriptor**
from your ABI + NatSpec, generate the **mandatory testsv2 fixtures** (the thing no existing tool does), run the full
registry CI locally, and open the PR.

## Why this exists

Every existing tool stops short of a mergeable submission:

| | generate | v2-native | uses NatSpec | testsv2 fixtures | local CI preflight | PR |
|---|---|---|---|---|---|---|
| `erc7730` (Ledger python CLI) | minimal scaffold | ❌ emits v1 | ❌ | ❌ | lint/format only | ❌ |
| Ledger web builder | UI | ❌ v1, dormant | ❌ | ❌ | ❌ | ❌ |
| Cyfrin `clearsig` | from Sourcify ABI | `--v2` alpha | ❌ | ❌ | ❌ | ❌ |
| Sourcify SDK / Rust SDK | ❌ (render-side) | ✅ | — | — | — | — |
| LI.FI's pipeline | ✅ but bespoke to their structs | ✅ | ❌ | ✅ hand-rolled | via registry CI | ✅ |
| **this** | ✅ | ✅ | ✅ intents from `@notice`/`@param` | ✅ generated | ✅ all six checks | ✅ scaffolded |

The registry **requires** a `testsv2/*.tests.json` per descriptor (CI hard-fails without it), and nothing generates them.
Both existing generators emit the old v1 shape with heuristic labels and no `intent`. NatSpec — the place developers
already wrote the human-readable meaning — is read by zero tools.

## How it plugs into Foundry

Forge has **no plugin system** ([foundry-rs/foundry#8266](https://github.com/foundry-rs/foundry/issues/8266) is still open),
so this follows the proven pattern (forge-deploy, foundry-devops): a standalone step after your normal deploy that reads
what forge already wrote —

1. `broadcast/<Script>.s.sol/<chainId>/run-latest.json` → deployed `(contractName, address, chainId)` per CREATE tx
2. `forge inspect <path>:<Name> abi|devdoc|userdoc --json` → ABI + NatSpec (zero config needed)
3. Sourcify API v2 → confirm verification (reviewers require verified source; lint silently skips field checks without it)

## Pipeline

```
forge script --broadcast --verify        (your normal deploy)
        │
        ▼
draft      ABI + NatSpec (+ verified source) → v2 descriptor
           @notice → intent · @param → labels · visibility judgment on every field
        ▼
tests      generate testsv2 fixtures: build/mine rawTx vectors, derive the
           dataProvider mock block, render expected output via the reference runners
        ▼
preflight  replicate registry CI locally: erc7730 lint --gha · check-jsonschema ·
           index collision · file names · sourcify test-runner · rust cs-test
        ▼
pr         fork/branch/PR to ethereum/clear-signing-erc7730-registry
           (stops at a reviewed draft; opening the PR is opt-in)
```

Compose, don't rebuild: lint/format = `erc7730`, rendering = the two pinned reference runners,
attestation hashing = `clearsig`. The agent's job is judgment (intent wording, field visibility) and glue.

## Status

Research done ([docs/RESEARCH.md](docs/RESEARCH.md)), pipeline being built. Near-term plan:

- [ ] hand-make 1–2 real descriptors end-to-end to harden the playbook (first candidate: a live contract with rich NatSpec)
- [ ] `skills/clear-signing/` — the agent skill (playbook + scripts), installable via `npx skills add`
- [ ] `preflight` script — all six registry CI checks runnable locally before pushing
- [ ] testsv2 generator — the missing piece for everyone
- [ ] wire into the Sourcify verification flow (descriptor drafted at verify time — with @kuzdogan)

## Related

[ERC-7730](https://eips.ethereum.org/EIPS/eip-7730) · [registry](https://github.com/ethereum/clear-signing-erc7730-registry) ·
[clearsigning.org](https://clearsigning.org) · [portdeveloper/clear-signing-helper](https://github.com/portdeveloper/clear-signing-helper)
(address-first playbook this complements — that one starts from a deployed address, this one starts from your Foundry workflow
and adds v2 + NatSpec + test generation)
