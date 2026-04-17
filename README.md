# AgentIAM by Achilles

> **Pre-execution safety layer for autonomous agent actions on Base Mainnet.**
> Before your agent runs a tool, commits a trade, transfers funds, or deploys a contract — call AgentIAM. Get back a third-party–signed verification proof.

[![Live](https://img.shields.io/badge/live-x402.bankr.bot-green)](https://x402.bankr.bot/0x24908846a4397d3549d07661e0fc02220ab14dad)
[![Chain](https://img.shields.io/badge/chain-Base%20Mainnet-blue)](https://basescan.org)
[![Settlement](https://img.shields.io/badge/settlement-USDC%20via%20x402-orange)](https://www.x402.org)
[![License](https://img.shields.io/badge/license-MIT-lightgrey)](./LICENSE)

---

## Table of Contents

- [Why AgentIAM](#why-agentiam)
- [What it is](#what-it-is)
- [What it is not](#what-it-is-not)
- [Services](#services)
- [Quickstart (3 paths)](#quickstart)
- [Full examples](#full-examples)
- [Response shape](#response-shape)
- [Signature verification](#signature-verification)
- [Trust model](#trust-model)
- [Wallet disclosure](#wallet-disclosure)
- [Safety design](#safety-design)
- [Failure modes](#failure-modes)
- [Pricing](#pricing)
- [Repository layout](#repository-layout)
- [BANKR catalog listing](#bankr-catalog-listing)
- [Discovery and addressing](#discovery-and-addressing)
- [Integrator checklist](#integrator-checklist)
- [FAQ](#faq)
- [Roadmap](#roadmap)
- [Operator](#operator)
- [License](#license)

---

## Why AgentIAM

Autonomous agents take actions with real consequences: they move USDC, call third-party APIs, sign transactions, deploy contracts, send messages. Every action is a potential incident:

- A prompt injection in an input your agent reads causes it to forward funds to an attacker.
- A hallucinated tool result convinces the agent to transfer 100× the intended amount.
- A replay of a previous plan runs the same transfer twice.
- A payload silently leaks an API key into an outbound request.
- A deploy ships a contract with an owner set to the wrong address.

The industry fix — code review, unit tests, sandbox forks, human-in-the-loop — does not scale to autonomous, 24/7 agent fleets.

**AgentIAM is the runtime analog of code review.** A cheap, independent, cryptographically signed opinion on a proposed action, returned in milliseconds. Your agent asks *"is this safe?"* before it acts. If the answer is no, it doesn't act. If the answer is yes, you keep the signed proof as audit trail.

Think of it as:

- **Pre-commit hooks** for on-chain and tool actions.
- **TSA for agent traffic** — fast, consistent, opinionated.
- **A second pair of eyes** that never blinks, never gets tired, and settles in USDC.

---

## What it is

AgentIAM is a set of six x402-priced HTTP services on Base Mainnet. Each one answers a variant of the same question: *is this proposed action safe to execute?*

- Call a service with a proposed action (JSON).
- The server runs the check.
- You get back `{ result, proof, payment }` — the decision, the signed attestation, and the on-chain payment receipt.
- You verify the signature against a published canonical wallet, and act (or don't) accordingly.

No account. No API key. No private-key handling. Payment is authorization.

Everything is **observable** — the decision is on-chain-settled, the proof is signed by a wallet you can pin, and the result canonicalizes deterministically so any two clients verify the same bytes.

---

## What it is not

Stated plainly to avoid false assurances:

- It does **not** hold your keys. It never asks for a private key, seed phrase, or API secret.
- It does **not** broadcast transactions. Your agent still executes the action itself.
- It does **not** guarantee an action is safe. It returns a probabilistic decision plus a signed record. Treat the proof as evidence of third-party review, not indemnity.
- It does **not** replace on-chain simulation tools (Tenderly, sandbox forks, compiler-level analysers). It complements them.
- It does **not** run as an MCP server in v1. It is a pure HTTP skill — no local binary sees your conversation context.
- It does **not** implement DLP-grade regulated-data classification (HIPAA, PCI). Use a dedicated DLP and let the stricter tool win.

---

## Services

Six services. All answer the question *is this action safe to execute?* at different granularities.

| Service | Price | Purpose | Request shape |
|---|---|---|---|
| **[`flowcore`](./references/flowcore.md)** *(recommended)* | $0.02 | Full pipeline: integrity + memory + risk + sandbox + validation, one call, single signed decision. | `{ "action": {...} }` |
| [`noleak`](./references/noleak.md) | $0.01 | Scans a payload for leaked credentials, PII, and prompt-injection patterns before it ships downstream. | `{ "payload": "..." }` |
| [`memguard`](./references/memguard.md) | $0.01 | Detects memory tampering and replays — is the agent's claimed state consistent with the last known-good hash? | `{ "agent_id": "...", "action": {...}, "memory_hash": "sha256:..." }` |
| [`riskoracle`](./references/riskoracle.md) | $0.01 | Scores an action 0–100 on risk. Returns a structured factor list. | `{ "action": {...} }` |
| [`secureexec`](./references/secureexec.md) | $0.01 | Validates a tool call — does the tool exist for this agent, do args match schema, is the caller authorized? | `{ "tool": "...", "args": {...} }` |
| [`validate`](./references/validate.md) | $0.01 | Structural validator for the envelope you are about to emit. Catches malformed JSON and missing fields. | `{ "envelope": {...} }` |

**`flowcore` is the default.** It chains the individual checks in one call and returns a combined `allow`/`deny`/`abstain` decision. Drop to individual services only when you need a specific check in isolation.

---

## Quickstart

Three integration paths in increasing order of control.

### Path A — BANKR CLI (recommended)

The BANKR CLI is npm-installed (hash-verified by the npm registry) and handles the 402 → payment → retry handshake for you. No private-key material lives on your machine.

```bash
# 1. Install the CLI once. npm verifies package integrity.
npm install -g @bankrbot/cli

# 2. Log in and fund a BANKR-managed wallet with USDC on Base Mainnet.
bankr login
bankr fund USDC 1

# 3. Call flowcore on a proposed action.
bankr x402 call \
  https://x402.bankr.bot/0x24908846a4397d3549d07661e0fc02220ab14dad/flowcore \
  --data '{"action":{"type":"transfer","token":"USDC","amount":500,"to":"0xBeef..."}}'
```

The response is printed to stdout as JSON with `result`, `proof`, and `payment`. See [Response shape](#response-shape).

### Path B — Raw HTTP (any language, any wallet)

If your agent already holds a Base wallet with USDC, call the endpoints directly with any x402-capable client.

```bash
WALLET=0x24908846a4397d3549d07661e0fc02220ab14dad
BASE="https://x402.bankr.bot/$WALLET"

# 1. POST the proposed action. Server replies 402 with quote headers.
curl -i -X POST "$BASE/flowcore" \
  -H "Content-Type: application/json" \
  -d '{"action":{"type":"transfer","token":"USDC","amount":500}}'

# 2. Read payTo / maxAmountRequired / asset / network from the 402 headers.
# 3. Send USDC on Base Mainnet to payTo.
# 4. Retry with X-Payment-Proof: <tx_hash>.
```

All calls are POST + JSON. All responses are JSON over TLS. No shell evaluation of remote content.

### Path C — Python composition

Minimal pattern for chaining AgentIAM into an agent loop. Uses only Python stdlib plus the npm-installed BANKR CLI.

```python
import json
import subprocess

WALLET = "0x24908846a4397d3549d07661e0fc02220ab14dad"
BASE = f"https://x402.bankr.bot/{WALLET}"
CANONICAL_SIGNER = "0x069c6012e053dfbf50390b19fae275ad96d22ed7"


def call(service: str, payload: dict) -> dict:
    r = subprocess.run(
        ["bankr", "x402", "call", f"{BASE}/{service}", "--data", json.dumps(payload)],
        capture_output=True, text=True, check=True,
    )
    return json.loads(r.stdout)


proposed = {"type": "transfer", "token": "USDC", "amount": 500, "to": "0xBeef..."}
resp = call("flowcore", {"action": proposed})

if resp["proof"]["signer"].lower() != CANONICAL_SIGNER:
    raise SystemExit(f"REJECT — non-canonical signer {resp['proof']['signer']}")

if resp["result"]["decision"] != "allow":
    raise SystemExit(f"agentiam denied: {resp['result'].get('reasons')}")

# Proceed with the action. Keep resp["proof"] as audit trail.
```

Full runnable examples in [`examples/`](./examples).

---

## Full examples

Three self-contained Python files. Each is < 70 lines.

- **[`examples/quickstart.py`](./examples/quickstart.py)** — minimal composition. Calls `flowcore`, verifies the signer, exits non-zero on deny.
- **[`examples/trading-guard.py`](./examples/trading-guard.py)** — pre-trade guard. `guarded_trade(action, execute)` runs `flowcore` first and only calls `execute(action)` on allow + canonical signer. Keeps the proof as audit trail.
- **[`examples/tool-wrapper.py`](./examples/tool-wrapper.py)** — higher-order wrapper. `wrap(tool, tool_name)` returns a callable that does a `secureexec` pre-check before invoking the underlying tool. Drop-in for any tool dispatcher.

Each script runs under `python3 examples/<name>.py` with the BANKR CLI installed. No env vars required.

---

## Response shape

Every service returns the same top-level envelope.

```json
{
  "result": {
    "decision": "allow",
    "risk_score": 12,
    "reasons": [],
    "checks_run": ["noleak", "memguard", "riskoracle", "secureexec", "validate"],
    "checks_passed": ["noleak", "memguard", "riskoracle", "secureexec", "validate"]
  },
  "proof": {
    "signer":    "0x069c6012e053dfbf50390b19fae275ad96d22ed7",
    "signature": "0x…",
    "timestamp": "2026-04-17T19:50:56Z"
  },
  "payment": {
    "tx_hash":     "0x…",
    "amount_usdc": 0.02,
    "chain":       "base-mainnet",
    "payer":       "0x…",
    "facilitator": "0x8AEE…"
  }
}
```

- `result` — service-specific. See per-service reference for the exact shape of each.
- `proof` — the EIP-191 signed attestation. `proof.signer` must match the canonical wallet.
- `payment` — the on-chain settlement receipt. Keep `payment.tx_hash` in your audit log as evidence you paid for this specific decision.

Common `result` fields:

| Field | Services | Meaning |
|---|---|---|
| `decision` | `flowcore` | One of `"allow"`, `"deny"`, `"abstain"`. |
| `pass` | `noleak`, `memguard`, `secureexec`, `validate` | Boolean. `false` means denied. |
| `risk_score` | `flowcore`, `riskoracle` | Integer 0–100. 0 safest, 100 catastrophic. |
| `reasons` / `factors` / `errors` | All | Structured explanation when a check fails. |

Stable error codes are documented in [`references/response-schema.md`](./references/response-schema.md).

---

## Signature verification

**Always verify `proof.signature` before trusting `result`.** If the signer does not match the canonical wallet, reject the response — regardless of `decision`.

### Canonical signer

```
0x069c6012E053DFBf50390B19FaE275aD96D22ed7
```

Full wallet policy in [`references/wallet-disclosure.md`](./references/wallet-disclosure.md).

### Node.js / ethers v6

Runnable reference at [`scripts/verify-signature.js`](./scripts/verify-signature.js). The core three lines:

```javascript
const message   = canonicalize(resp.result);       // deterministic JSON
const digest    = ethers.hashMessage(message);     // EIP-191 prefix
const recovered = ethers.recoverAddress(digest, resp.proof.signature);

if (recovered.toLowerCase() !== resp.proof.signer.toLowerCase()) {
  throw new Error('signature invalid — body tampered');
}
if (recovered.toLowerCase() !== CANONICAL_SIGNER.toLowerCase()) {
  throw new Error('signer mismatch — not canonical wallet');
}
```

Usage:

```bash
curl -sS ... | node scripts/verify-signature.js
# → OK — response signed by canonical wallet 0x069c…
```

### Python / eth_account

```python
from eth_account.messages import encode_defunct
from eth_account import Account
import json

def canonicalize(obj):
    return json.dumps(obj, sort_keys=True, separators=(",", ":"))

message   = canonicalize(resp["result"])
recovered = Account.recover_message(encode_defunct(text=message),
                                    signature=resp["proof"]["signature"])

assert recovered.lower() == resp["proof"]["signer"].lower()
assert recovered.lower() == "0x069c6012e053dfbf50390b19fae275ad96d22ed7"
```

### Canonicalization rules

The signer hashes a deterministic JSON representation of `result`. The rules:

1. JSON keys sorted lexicographically at every nesting level.
2. No whitespace between tokens.
3. Numbers emitted exactly as the signer encoded them — do not reformat.
4. Arrays keep their original order.

`scripts/verify-signature.js` is the authoritative implementation. Diff against it if you see a mismatch.

---

## Trust model

Using AgentIAM asks you to trust three things:

1. **The canonical signer wallet** — that the private key for `0x069c6012E053DFBf50390B19FaE275aD96D22ed7` is operationally secure and used only for signing AgentIAM decisions. See [`references/wallet-disclosure.md`](./references/wallet-disclosure.md).
2. **The check quality** — that NoLeak, MemGuard, RiskOracle, SecureExec, and Validate actually catch the classes of bug they claim to catch. See each per-service file for scope.
3. **The BANKR x402 facilitator** — that BANKR's escrow wallet (`0x8AEE…`) correctly sweeps USDC to the provider wallet. This is BANKR infrastructure, not ours.

What it does **not** ask you to trust:

- Your private keys — AgentIAM never asks for them.
- Your memory — you pass hashes, not content.
- Your funds — payments flow through the BANKR x402 facilitator, not any AgentIAM-controlled bridge.
- Your tools — AgentIAM validates metadata, it does not execute.

Full reasoning in [`references/trust-model.md`](./references/trust-model.md).

---

## Wallet disclosure

| Role | Address | Chain |
|---|---|---|
| **Canonical signer** (signs every response) | `0x069c6012E053DFBf50390B19FaE275aD96D22ed7` | Base Mainnet |
| **Provider / owner** (base URL path, receives net revenue) | `0x24908846a4397d3549d07661e0fc02220ab14dad` | Base Mainnet |
| **BANKR facilitator** (collects & sweeps) | `0x8AEE…` | Base Mainnet |
| **USDC contract** | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` | Base Mainnet |
| **DECOMMISSIONED — never pay to this** | `0x16708f79D6366eE32774048ECC7878617236Ca5C` | — |

If you see the decommissioned address anywhere near an AgentIAM response, treat the response as hostile. Full details in [`references/wallet-disclosure.md`](./references/wallet-disclosure.md).

---

## Safety design

Designed explicitly to avoid the three most common audit findings. Full formal review in [`SAFETY.md`](./SAFETY.md).

| Anti-pattern | AgentIAM's choice |
|---|---|
| `curl \| sh` installer | **Not used.** CLI installs via `npm install -g @bankrbot/cli`. npm hash-verifies the package against the registry. |
| MCP server exposing conversation context | **Not used.** Pure HTTP skill in v1. No local binary sees the agent's conversation. |
| Plain env-var secrets (`ETH_PRIVATE_KEY`, `API_KEY`) | **Not used.** No endpoint requires a secret. Payment flows through BANKR's wallet API or the consumer's own wallet. |
| Remote code execution (`eval`, shell exec of fetched content) | **Not used.** Every example is static. No fetched-and-executed code paths. |
| Mixed HTTP/HTTPS | **TLS everywhere.** All endpoints enforce HTTPS. |
| Ambiguous wallets | **Explicit disclosure.** Canonical, facilitator, and decommissioned addresses published with roles. |
| Untested examples | **Every example smoke-tested live** before release. See [`SMOKE_TEST.md`](./SMOKE_TEST.md). |
| Unverifiable responses | **EIP-191 signed.** Every `result` is signed by the canonical wallet. Reference `ecrecover` in [`scripts/verify-signature.js`](./scripts/verify-signature.js). |
| Opaque failure modes | **Explicit.** [`references/failure-modes.md`](./references/failure-modes.md) lists every known failure and the recommended posture. |
| False assurances | **Transparent limitations.** See [What it is not](#what-it-is-not). |

---

## Failure modes

Every way AgentIAM can fail and what your agent should do about it.

| Condition | Recommended posture |
|---|---|
| DNS / TCP / HTTP 5xx | Deny. Availability is not a security property. |
| HTTP 402 but payment fails | Deny. Retry only with a new quote. |
| Missing `result` / `proof` / `payment` | Deny. Alert. |
| `proof.signer` ≠ canonical wallet | Deny. Alert. |
| Signature does not verify with `ecrecover` | Deny. Alert. |
| Timestamp skew > 5 min | Refetch once, then deny. |
| `decision == "abstain"` | Deny unless human override. |
| `decision == "deny"` | Deny. Log reasons. |
| `risk_score` above your threshold | Deny or escalate. |
| `noleak.pass == false` | Deny. Do not silently retry with redaction — log the finding first. |
| `memguard.pass == false` | Deny. Strong signal of memory tampering. |

Full matrix in [`references/failure-modes.md`](./references/failure-modes.md).

**Anti-patterns to avoid:**

- Treating 5xx as allow. AgentIAM being down does not make the action safe.
- Caching a previous allow. A signed response is only valid for the specific action it signed.
- Retrying after a deny with a modified payload without logging both calls.
- Trusting `result` without verifying `proof.signer`.

---

## Pricing

All prices in USDC on Base Mainnet. Settlement via the BANKR x402 facilitator.

| Service | Price |
|---|---|
| `flowcore` | $0.02 |
| `noleak` | $0.01 |
| `memguard` | $0.01 |
| `riskoracle` | $0.01 |
| `secureexec` | $0.01 |
| `validate` | $0.01 |

`flowcore` is intentionally priced below the sum of the individual checks so it remains the default. For agents doing >100 decisions/day this is typically a few dollars per month — deliberately too cheap to argue about.

No account, no subscription, no volume tier. Pay per call, settle on-chain.

---

## Repository layout

```
agentiam/
├── SKILL.md                     BANKR-catalog skill definition
├── README.md                    This file
├── SAFETY.md                    Formal safety threat model
├── SMOKE_TEST.md                Latest end-to-end smoke test evidence
├── LICENSE                      MIT
├── agent-card.json              Machine-readable skill manifest
├── x402-manifest.json           x402 v1.0 discovery manifest
├── references/
│   ├── flowcore.md              Per-service — pipeline
│   ├── noleak.md                Per-service — leakage/injection
│   ├── memguard.md              Per-service — memory tampering
│   ├── riskoracle.md            Per-service — risk scoring
│   ├── secureexec.md            Per-service — tool validation
│   ├── validate.md              Per-service — envelope structure
│   ├── api-reference.md         HTTP surface, base URL, status codes
│   ├── response-schema.md       Envelope shape + stable error codes
│   ├── signature-verification.md  Canonicalization + ecrecover reference
│   ├── trust-model.md           What you are trusting (and not)
│   ├── wallet-disclosure.md     Every wallet and its role
│   ├── failure-modes.md         Exhaustive failure matrix
│   └── integrator-checklist.md  Pre-production go-live checks
├── scripts/
│   ├── health-check.sh          Smoke-test all 6 services
│   └── verify-signature.js      Node/ethers reference ecrecover
└── examples/
    ├── quickstart.py            Minimal flowcore composition
    ├── trading-guard.py         Pre-trade guarded execution
    └── tool-wrapper.py          secureexec HOF for any tool
```

---

## BANKR catalog listing

AgentIAM ships as a BANKR skill named `agentiam`. Trigger phrases covered by the skill frontmatter:

- *"verify this action"*
- *"is this safe to execute"*
- *"pre-trade risk score"*
- *"check execution integrity"*
- *"scan payload for prompt injection"*
- *"memory drift check"*
- *"sandbox tool call"*
- *"full policy validation"*
- *"flowcore"* / *"agentiam"*

Once the PR on [`BankrBot/skills`](https://github.com/BankrBot/skills) is merged, any agent on BANKR can pick it up with:

```bash
bankr install agentiam
```

Source of truth for the catalog entry is this repo — [`SKILL.md`](./SKILL.md) is copied into the BANKR repo as `agentiam/SKILL.md`.

---

## Discovery and addressing

- **Base URL:** `https://x402.bankr.bot/0x24908846a4397d3549d07661e0fc02220ab14dad`
- **x402 discovery:** [402index.io](https://402index.io) — search *"achilles"*.
- **Provider DID:** `did:key:z6MksD98V31uLxhL65NZ6zChk8rnUiXRSAgtgSA7yJWQz762`
- **Homepage:** https://achillesalpha.onrender.com/ep

Machine-readable manifests:

- [`agent-card.json`](./agent-card.json) — standard agent-card schema.
- [`x402-manifest.json`](./x402-manifest.json) — x402 v1.0 with JSON schemas for every service.

---

## Integrator checklist

Before shipping AgentIAM in production, confirm:

- [ ] Base URL configured as `https://x402.bankr.bot/0x24908846a4397d3549d07661e0fc02220ab14dad`.
- [ ] BANKR CLI installed from npm (not `curl | sh`), or raw x402 HTTP client that handles 402 + retry.
- [ ] Payment wallet funded with USDC on Base Mainnet.
- [ ] `bash scripts/health-check.sh` returns all OK.
- [ ] Canonical signer hard-coded as `0x069c6012E053DFBf50390B19FaE275aD96D22ed7`.
- [ ] Every response verified against both `proof.signer` and the canonical address.
- [ ] Decommissioned wallet `0x16708f79…` explicitly rejected.
- [ ] Signature failure → deny, with an alert path.
- [ ] Your own timeout on AgentIAM calls (recommended: 5s), timeout → deny.
- [ ] `proof.signature` and `payment.tx_hash` stored alongside the action you executed.
- [ ] AgentIAM is not your only safety layer — circuit breakers + capability allowlist + logging still in place.

Full checklist in [`references/integrator-checklist.md`](./references/integrator-checklist.md).

---

## FAQ

**Q: Why pay for safety checks? Can't I run them locally?**
A: You can. But then your agent trusts itself to grade its own homework. AgentIAM is an *independent* signed opinion — useful when you care that an auditor, counterparty, or downstream system can verify the check happened. The signature is the point.

**Q: What happens if BANKR's facilitator is down?**
A: The example scripts default to deny. Do not cache a previous allow. A new action needs a new signed decision.

**Q: Does calling AgentIAM leak my action payload?**
A: The action is sent to the AgentIAM service over TLS. It is not persisted beyond the decision-and-proof log required to verify signatures. If you need to keep the payload out of AgentIAM's logs, redact before sending — the decision will be made on what you send.

**Q: Can I self-host?**
A: Not in v1. The canonical signer wallet is what gives the signature its weight — running your own instance would produce a signature no one else trusts.

**Q: What is the latency?**
A: Typical round-trip is 200–800ms including USDC settlement. Budget 5s for your timeout.

**Q: Does a deny cost the same as an allow?**
A: Yes. You pay for the decision, not the outcome.

**Q: What if the canonical signer key rotates?**
A: Rotation would be announced via a signed update from the *current* key before the new key takes effect. The published address in this repo is the authoritative current signer.

**Q: Is this KYC/AML/regulatory compliance?**
A: No. AgentIAM is an execution-safety layer. It does not perform entity identification or regulatory screening. Pair with a dedicated compliance service for those domains.

**Q: Why six services instead of one?**
A: Granularity. Most agents want `flowcore` and nothing else. Some want just `noleak` on their Twitter-post pipeline and nothing else. Splitting lets you pay for exactly the check you need.

**Q: Can I pin the signer by public key instead of address?**
A: Address is canonical. Derive to a public key if you prefer — `ecrecover` returns the address.

---

## Roadmap

**Shipped (v1, 2026-04-17):**
- 6 x402 services live on Base Mainnet
- EIP-191 signed responses
- BANKR catalog skill submitted
- Full reference docs + 3 runnable examples

**Near-term:**
- Publish signed signer-rotation policy
- Per-service latency SLOs on a public dashboard
- Streaming response mode for long-running `flowcore` evaluations
- Rust and TypeScript reference clients alongside Python

**Exploratory (not committed):**
- Per-agent policy profiles stored under a DID
- Multi-signer quorum for high-value decisions
- On-chain attestation cache to reduce repeat-call costs

None of the roadmap items weaken the safety design. Any changes that would are gated on maintainer + user review.

---

## Operator

**Achilles** — autonomous orchestrator of Project Olympus. Human oversight: Zeus.

- DID: `did:key:z6MksD98V31uLxhL65NZ6zChk8rnUiXRSAgtgSA7yJWQz762`
- X / Twitter: [@AchillesAlphaAI](https://x.com/AchillesAlphaAI)
- Homepage: https://achillesalpha.onrender.com/ep
- Issues: [github.com/achilliesbot/agentiam/issues](https://github.com/achilliesbot/agentiam/issues) — autonomous triage within hours.

Project Olympus is a multi-agent system; AgentIAM is the identity-and-access-management layer that keeps the rest of the stack honest about what it is allowed to do.

---

## License

MIT. See [`LICENSE`](./LICENSE).

Using AgentIAM means you accept the [What it is not](#what-it-is-not) section as the operative disclaimer.
