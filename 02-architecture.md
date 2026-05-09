# Rayachain Cross-Chain Architecture: Design & Capabilities

**Technical depth:** Advanced | **Audience:** Protocol engineers, risk researchers  
**Complements:** [OmniRisk System Design](LINK_NEEDED) | [API Reference](04-api-examples.md)

---

## What Rayachain Does

Rayachain is the cross-chain layer of OmniRisk that handles two capabilities:

1. **Token Transfer** — Move tokens across six blockchains using LayerZero's OFT protocol with quote → build → submit flow
2. **Risk Broadcast** — Publish OmniRisk's risk intelligence (finality, liquidity, operator health, economic signals) to destination chains via LayerZero

The system is built on a **hub-and-spoke topology** with Solana as the hub. All cross-chain messages flow through Solana, simplifying routing while creating a single point of architectural leverage.

**Current deployment:**
- **Testnet:** Phase 1A (Ethereum, Base), 1B (Arbitrum), 1C (Polygon, BSC) — all active
- **Mainnet:** Configuration exists, not yet activated (feature gate pending)

---

## Architecture Overview

```
OmniRisk Platform
├─ Prediction Engine (heuristic + statistical)
│  └─ 66k historical samples, regime detection, tail-risk analysis
├─ Market Intelligence Engine (12+ news + social feeds)
│  └─ Lexicon-scored sentiment + LLM enrichment (optional)
└─ Agentic AI (LangGraph, 13-phase, deterministic-first)
   └─ Multi-provider routing (OpenAI, Anthropic, Gemini, local)

          ↓

Rayachain Layer (Package: broadcast-phase1.ts)
├─ Transfer Capability
│  └─ Quote (LayerZero fee query)
│  └─ Build (construct signed tx)
│  └─ Submit (user signs + relays)
│
└─ Broadcast Capability
   └─ Plan (eligibility check + lane readiness)
   └─ Quote (per-destination fee)
   └─ Send (Solana sidecar OR LayerZero for EVM)
   └─ Track (per-destination delivery status)

          ↓

LayerZero OFT Protocol
├─ Solana (Hub) — all messages route through
└─ Destinations:
   ├─ Ethereum (1A)
   ├─ Arbitrum (1B)
   ├─ Base (1A)
   ├─ Polygon (1C)
   └─ BSC (1C)
```

---

## Core Components

### 1. Transfer Engine

Handles quote → build → submit flow for cross-chain token moves.

**Endpoints:**
- `POST /v1/rayachain/transfers/quote` — Get LayerZero fee estimate
- `POST /v1/rayachain/transfers/build` — Construct a signed transfer
- `GET /v1/rayachain/transfers/:id/timeline` — Track status across chains

**Flow:**
```
User: Quote for 100 USDC, Ethereum → Arbitrum
  ↓
API: Query LayerZero for fee (gas on both chains)
  ↓
Response: 100 USDC + 0.05 ETH fee = 100.001 USDC equivalent
  ↓
User: Build a transfer with that quote
  ↓
API: Construct transaction (unsigned)
  ↓
Response: Tx data + signature requirements
  ↓
User: Sign locally + relay to Ethereum RPC
  ↓
LayerZero relayer: Picks up message from Ethereum
  ↓
Destination: Arbitrum receives 100 USDC (arrives in 2–12 minutes depending on LayerZero config)
```

The API never holds private keys or submits on behalf of users. Status tracking is pull-based via the timeline endpoint.

### 2. Broadcast Capability

Sends OmniRisk risk intelligence to multiple chains in a single operation.

**Endpoints:**
- `POST /v1/rayachain/broadcast/plan` — Validate broadcast eligibility + lane readiness
- `POST /v1/rayachain/broadcast/quote` — Per-destination fee quote
- `POST /v1/rayachain/broadcast/send` — Submit broadcast
- `GET /v1/rayachain/broadcast/:id/destination/:chain` — Track per-destination state

**Flow:**
```
OmniRisk: "Risk for Polygon bridge = 65/100. Send to ETH, Arbitrum, Base."
  ↓
Rayachain Plan: Check if publisher is authorized, lanes are healthy
  ↓
Rayachain Quote: Get fees for each destination
  ↓
Rayachain Send:
  ├─ Solana sidecar execution (broadcasts to Solana program)
  └─ LayerZero delivery to EVM chains (Ethereum, Arbitrum, Base, Polygon, BSC)
  ↓
Result: Risk verdict available on-chain on each destination within 2–12 minutes
```

Unlike token transfers (which are pull-initiated by users), broadcasts are push-initiated by OmniRisk operators. Policy rules control which addresses can broadcast to which chains.

### 3. Solana Sidecar

Handles Solana-specific broadcast transactions because Solana's execution model (async, non-EVM) requires custom handling.

The sidecar is a separate HTTP service communicating with the main API via `solana-broadcast-executor.ts`. It:
- Receives broadcast payloads from the main API
- Constructs Solana transactions targeting the Rayachain program
- Submits to the Solana network
- Returns tx signatures for tracking

This decoupling keeps Solana complexity separate from the main service, preventing Solana dependencies from affecting EVM operations.

### 4. LayerZero OFT Protocol Integration

Both token transfers and risk broadcasts use LayerZero's OFT standard for cross-chain messaging.

**Why LayerZero?**
- Ultra-light client model (minimal trust assumptions)
- Proven adoption (100s of billions in value)
- Support for all six target chains
- Flexible fee/timeout configuration per route

**Key properties:**
- **Finality:** 2–12 minutes depending on destination chain's finality (Solana is fastest, Ethereum slowest)
- **Atomicity:** Individual message delivery is all-or-nothing (no partial failures)
- **Fallback:** If a relayer fails, LayerZero's protocol allows retriable submission

---

## Data Flow: Risk Broadcast Example

```
Step 1: OmniRisk Prediction Engine
  Analyzes 8 chains + 12 news feeds + social sentiment + agentic investigation
  Outputs: Risk score + confidence + anomalies
  Example: "Polygon bridge = 65/100 (medium risk). Liquidity concern. Operator key rotation detected."

Step 2: Rayachain Broadcast Layer
  Receives from OmniRisk: Risk verdict object
  Validates: Is publisher authorized? Are lanes ready?
  
  If valid:
    ├─ Solana sidecar: Submit tx to Rayachain program
    ├─ LayerZero EVM: Queue message to Ethereum endpoint
    ├─ LayerZero EVM: Queue message to Arbitrum endpoint
    ├─ LayerZero EVM: Queue message to Base endpoint
    └─ ... (repeat for Polygon, BSC)

Step 3: Cross-Chain Relay
  Relayers listen to Solana for outbound messages
  Relayers listen to Ethereum/Arbitrum/Base for incoming LayerZero messages
  Each destination chain receives the risk verdict
  Latency: 2–12 min per chain

Step 4: On-Chain Consumption
  Protocols can query: "What's the current risk for Polygon?"
  Answer comes from on-chain risk oracle contract
  Smart contracts can gate operations: "If Polygon risk > 70, pause this route"
```

---

## API Response Structure

All endpoints return a standard envelope:

```json
{
  "data": { /* endpoint-specific payload */ },
  "meta": {
    "requestId": "req_abc123",
    "timestamp": "2026-05-09T16:40:00Z",
    "version": "v1",
    "freshness": "live" // or "cached" on some endpoints
  },
  "error": null // present if the request failed
}
```

List endpoints extend `meta` with pagination:
```json
{
  "meta": {
    "page": 1,
    "pageSize": 20,
    "total": 47,
    "appliedFilters": { "status": "delivered" },
    "sortBy": "createdAt",
    "sortOrder": "desc"
  }
}
```

---

## Design Decisions

### Hub-and-Spoke with Solana

**Why Solana as hub?**
- **Finality speed.** Solana confirms in ~400ms. Ethereum takes 2–5 minutes. By routing through Solana, we can confirm delivery quickly and then relay outbound with less uncertainty.
- **Cost.** Solana transactions are $0.00025. This is negligible for message routing overhead.
- **Decoupling.** Protocols on other chains don't depend on Solana for token transfers; Solana is just a relay for message coordination.

**Trade-off:** If Solana halts (like the May 2021 outage), broadcast delivery halts. This is an intentional architectural choice: the alternative (peer-to-peer routing without a hub) is more complex and slower.

### Sidecar Pattern for Solana

Solana's async execution, commitment levels, and transaction construction differ significantly from EVM.  Rather than embedding Solana knowledge in the main API, it's isolated in a sidecar service. This keeps the main service EVM-focused and lets the sidecar optimize for Solana primitives.

### Policy Enforcement at Plan Time

Broadcast eligibility (who can broadcast, to which chains) is checked at `POST /broadcast/plan`, not at send time. This prevents wasted gas on ineligible operations and gives users early feedback on whether their broadcast is permitted.

### REST-Only API (for now)

Currently, all endpoints are request-response (HTTP POST/GET). WebSocket support is defined in CDK but not yet activated. This simplifies deploymentand testing while still allowing high-frequency polling if needed.

---

## Deployment Topology

Rayachain runs on **AWS** with a **Blue/Green topology**:

- **Blue (us-east-1):** Primary, 100% traffic
- **Green (eu-west-1):** Standby, 0% traffic (ready for failover)

Rollout is controlled by feature flags in `packages/shared/broadcast-phase1.ts`, allowing gradual activation of:
- New chains
- New risk broadcast categories
- New transfer types

**Current phase:** Testnet Phase 1A/1B/1C. Mainnet activation pending board approval.

---

## Roadmap

**Q2 2026 (Current):** Testnet transfer + broadcast on 5 EVM chains

**Q3 2026:** Mainnet activation. First protocols integrate risk gating.

**Q4 2026:** Native risk oracle contract deployed on all 6 chains. Direct smart contract querying enabled.

**2027+:** Governance token (RYA) launch. DAO controls broadcast policy and fee allocation.

---

## Further Reading

- **[API Reference](04-api-examples.md)** — Detailed endpoint documentation
- **[Getting Started](01-getting-started.md)** — Quick-start for token transfers
- **[OmniRisk System Design](LINK_NEEDED)** — How risk signals are generated
- **[Deployment Logs](LINK_NEEDED)** — Current testnet status per chain

---

**Resources & Links:**
- [OmniRisk Platform](https://omnirisk.io) — Risk intelligence for 8 blockchains
- [Rayachain](https://rayachain.omnirisk.io/) — Cross-chain token transfer + risk broadcast layer

**Questions?** Open an issue on [GitHub](github.com/baz2024/risk-intel) or ask in [Discord](LINK_NEEDED).
