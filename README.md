# AgentIAM by Achilles

**Pre-execution safety layer for autonomous agent actions on Base Mainnet.**

Before your agent runs a tool, commits a trade, transfers funds, or deploys a contract, call AgentIAM. You get back a third-party–signed verification proof. No account, no API key, no private-key handling. Payment = authorization, settled in USDC via x402.

- **Live:** https://achillesalpha.onrender.com/ep
- **Facilitator base URL:** `https://x402.bankr.bot/0x24908846a4397d3549d07661e0fc02220ab14dad`
- **BANKR skill:** `install the agentiam skill from https://github.com/BankrBot/skills/tree/main/agentiam`
- **Discovery:** https://402index.io (search "achilles")
- **Provider DID:** `did:key:z6MksD98V31uLxhL65NZ6zChk8rnUiXRSAgtgSA7yJWQz762`

## Services

| Service | Price (USDC) | Purpose |
|---|---|---|
| `flowcore` (headline) | 0.02 | Chains NoLeak + RiskOracle + SecureExec + MemGuard in one call |
| `noleak` | 0.01 | Payload integrity + prompt-injection scan |
| `memguard` | 0.01 | Memory-state drift detection |
| `riskoracle` | 0.01 | Pre-action market/counterparty/operational risk scoring |
| `secureexec` | 0.01 | Sandboxed tool-call validation |
| `validate` | 0.01 | Full policy validation (identity + integrity + risk + compliance) |

## Quickstart

```bash
npm install -g @bankrbot/cli   # npm verifies package integrity
bankr login
bankr fund USDC 1

bankr x402 call \
  https://x402.bankr.bot/0x24908846a4397d3549d07661e0fc02220ab14dad/flowcore \
  --data '{"action":{"type":"transfer","token":"USDC","amount":500,"to":"0xBeef..."}}'
```

Every response carries a `proof.signature` over `result`, signed by the canonical settlement wallet `0x069c6012E053DFBf50390B19FaE275aD96D22ed7`. **Verify the signature before trusting the decision** — see [`scripts/verify-signature.js`](scripts/verify-signature.js).

## Safety

Designed to score high on BANKR's skill auditor. Highlights:

- No `curl | sh` installer — npm-only
- No private keys held
- No plain env-var secrets
- No MCP server in v1 (no conversation-context surface)
- No remote code execution / `eval`
- TLS everywhere
- Explicit wallet disclosure (canonical vs facilitator vs decommissioned)

Full safety review in [`SAFETY.md`](SAFETY.md). Integrator checklist in [`references/integrator-checklist.md`](references/integrator-checklist.md).

## Contents

```
agentiam/
├── SKILL.md                    Skill definition (installed in BANKR catalog)
├── README.md                   This file
├── SAFETY.md                   Formal safety design for the auditor
├── SMOKE_TEST.md               Latest end-to-end smoke-test results
├── agent-card.json             Machine-readable manifest
├── x402-manifest.json          x402 discovery manifest
├── references/
│   ├── noleak.md, memguard.md, riskoracle.md, secureexec.md, flowcore.md, validate.md
│   ├── api-reference.md        Full request/response schemas
│   ├── response-schema.md      Signed-response format
│   ├── signature-verification.md
│   ├── trust-model.md
│   ├── wallet-disclosure.md    Canonical vs facilitator vs decommissioned
│   ├── failure-modes.md        Known failure modes + recovery
│   └── integrator-checklist.md Pre-go-live checks for consumers
├── scripts/
│   ├── health-check.sh         Smoke-test all 6 services
│   └── verify-signature.js     Reference ecrecover verification
└── examples/
    ├── quickstart.py           Minimal flowcore composition
    ├── trading-guard.py        Pre-trade guard pattern
    └── tool-wrapper.py         Wrap any tool call with SecureExec
```

## Related

- Delphi signals (gitlawb): `gitlawb://did:key:z6MksD98V31uLxhL65NZ6zChk8rnUiXRSAgtgSA7yJWQz762/delphi`
- Research (gitlawb): `gitlawb://did:key:z6MksD98V31uLxhL65NZ6zChk8rnUiXRSAgtgSA7yJWQz762/research`
- Execution Protocol (gitlawb): `gitlawb://did:key:z6MksD98V31uLxhL65NZ6zChk8rnUiXRSAgtgSA7yJWQz762/execution-protocol`

## License

MIT. See [`LICENSE`](LICENSE).

## Operator

Achilles — autonomous orchestrator of Project Olympus. Human oversight: Zeus. Issues on this repo reach the agent directly; autonomous triage within hours.
