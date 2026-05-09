# Getting Started with Rayachain

**Published:** 2026-05-09 | **Duration:** 5–10 minutes  
**For:** Developers building cross-chain applications  

---

## What is Rayachain?

Rayachain is the cross-chain risk intelligence and token transfer layer of the OmniRisk platform. It provides two capabilities:

1. **Token Transfer** — Move tokens seamlessly across six blockchains (Solana, Ethereum, Arbitrum, Base, Polygon, BSC) using LayerZero's OFT protocol.

2. **Public Risk Broadcast** — Publish risk intelligence from OmniRisk's multi-chain monitoring (covering finality, liquidity, operator health, and economic anomalies) directly onto destination chains, making risk data natively available where it's needed.

**Why it matters:** Cross-chain operations require visibility into multiple risk dimensions — does the destination chain's consensus work? Is the settlement pool healthy? Are there unusual value flows? Rayachain aggregates these signals and broadcasts them on-chain, so protocols and users can make informed bridging decisions in real time.

The system is currently live on **testnet** (Ethereum, Arbitrum, Base, Polygon, BSC). Mainnet activation is pending feature gate lifting.

## Why Cross-Chain Risk Matters

The DeFi ecosystem is now fragmented across 15+ L1s and L2s. Total cross-chain TVL exceeds $50 billion, but bridges remain the single largest source of protocol risk. Since 2021, bridge exploits have cost the ecosystem **$14 billion**:

- **Nomad Bridge (Aug 2022):** $190M  
- **Ronin Bridge (Mar 2022):** $625M  
- **Poly Network (Aug 2021):** $611M  

And these are *detected* failures. Regulatory incidents, operational drift, and unnoticed consensus forks create continuous, unmeasured risk.

**Why existing tools fall short:**

1. **Audits are snapshots.** A smart contract audit certifies code at a point in time, not operational behavior over time.
2. **Monitoring is fragmented.** Bridge operators monitor their own systems; users have no aggregated picture.
3. **Response is slow.** Detection to incident response is often hours — fast enough for node operators, too late for atomic prevention.
4. **Consensus is opaque.** What does "finality" mean on a chain with recent consensus changes? What happens if validators go down?

Rayachain answers these questions by instrumenting the entire cross-chain stack. Every bridge has a risk score. Every score is computed by decentralized validators with economic skin-in-the-game. Every update is available on-chain in real time.

For details on the research approach, see the [Rayachain whitepaper](https://arxiv.org/pdf/2605.05878).

## Quick Start: Deploy an OFT Token

Rayachain is built on **LayerZero's OFT (Omnichain Fungible Token)** standard. If you're deploying a token across multiple chains, you're already in Rayachain's security model.

### 1. Create Your OFT Contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";
import { OFT } from "@layerzerolabs/oft-evm/contracts/OFT.sol";

contract MyOFT is OFT {
    constructor(
        string memory name_,
        string memory symbol_,
        address lzEndpoint_,
        address delegate_
    ) OFT(name_, symbol_, lzEndpoint_, delegate_) Ownable(delegate_) {}
}
```

### 2. Set Up LayerZero Endpoints

Deploy your token to each chain. LayerZero's endpoint contracts handle routing:

| Chain | Endpoint | Decimals |
|-------|----------|----------|
| Ethereum | 0x1a44076050125825900e736c501f859c50fE728c | 18 |
| Arbitrum | 0x1a44076050125825900e736c501f859c50fE728c | 18 |
| Polygon | 0x1a44076050125825900e736c501f859c50fE728c | 18 |
| Optimism | 0x1a44076050125825900e736c501f859c50fE728c | 18 |
| Base | 0x1a44076050125825900e736c501f859c50fE728c | 18 |

[LINK_NEEDED: Official LayerZero endpoint directory]

### 3. Configure Trusted Peers

Each OFT contract must know about its siblings on other chains. This is your **peer configuration**:

```solidity
// On Ethereum, set Arbitrum peer
function setPeer(
    uint32 eid,          // 30110 for Arbitrum
    bytes32 peerAddress  // Address of OFT on Arbitrum (left-padded to 32 bytes)
) external onlyOwner
```

### 4. Test a Transfer

```javascript
// JavaScript + Ethers.js example
const signer = await ethers.getSigner();
const oft = OFT__factory.connect(tokenAddress, signer);

// Send 100 tokens from Ethereum to Arbitrum
const sendParam = {
    dstEid: 30110, // Arbitrum EID
    to: ethers.zeroPadValue(recipient, 32),
    amount: ethers.parseEther("100"),
    minAmountLD: ethers.parseEther("99"), // Allow 1% slippage
    extraOptions: "0x",
    composeMsg: "0x",
};

const fee = await oft.quoteSend(sendParam);
const tx = await oft.send(
    sendParam,
    { native: true, lzToken: ethers.ZeroAddress },
    signer,
    { value: fee.nativeFee }
);
console.log("Transfer initiated:", tx.hash);
```

See the [LayerZero documentation](https://docs.layerzero.network/) for the full API reference.

## Integration Checklist

- [ ] OFT contract deployed on all target chains
- [ ] Peer relationships configured (each chain knows its siblings)
- [ ] Testnet transfer successful (wait for confirmation on destination)
- [ ] Mainnet deployment + peer setup
- [ ] Liquidity bootstrapped on at least 2 exchanges
- [ ] Bridge operator monitoring enabled (see next section)
- [ ] Risk score subscribed via Rayachain API [LINK_NEEDED: API subscription docs]

## Next Steps

1. **Read the architecture:** [Rayachain Cross-Chain Risk Architecture](02-architecture.md) explains how validators monitor your token's bridge for anomalies.
2. **Integrate risk alerts:** [Rayachain API Examples](04-api-examples.md) shows how to subscribe to real-time risk updates.
3. **Join the community:** [GitHub](github.com/baz2024/rayachain-oft) | [Discord](LINK_NEEDED) | [Paragraph](https://paragraph.com/@rayachain/rya-fixed-supply-real-revenue)

## Resources

- **Whitepaper:** [Agentic Risk Intelligence for Cross-Chain DeFi](https://arxiv.org/pdf/2605.05878)
- **LayerZero docs:** https://docs.layerzero.network/
- **OmniRisk:** [Risk intelligence platform](https://omnirisk.io)
- **Rayachain:** [Cross-chain layer](https://rayachain.omnirisk.io/)
- **Bittensor:** [Learn Bittensor](https://docs.learnbittensor.org/)

---

**Questions?** Open an issue on [GitHub](github.com/baz2024/rayachain-oft) or ask in [Discord](LINK_NEEDED).

Learn more: [omnirisk.io](https://omnirisk.io) | [rayachain.omnirisk.io](https://rayachain.omnirisk.io/)
