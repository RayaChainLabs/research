# Who Uses Rayachain: Real-World Integration Cases

**For:** Protocol teams, bridge operators, institutional stakeholders  
**Duration:** 10–15 minutes  
**Complements:** [Architecture](02-architecture.md) | [API Examples](04-api-examples.md)

---

## Overview

Rayachain risk scores are consumed by protocols across four use cases. Each case involves different participants and creates different value.

```
                      Rayachain Risk Scores
                              │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
   1. Bridge Gates         2. Liquidity Guard      3. Insurance Pricing
   (protocol-level)        (market-making)         (risk premium)
        │                       │                       │
        ▼                       ▼                       ▼
    Polygon,              Uniswap V4             Nexus Mutual,
    Arbitrum,             Curve,                 Lido Community
    Optimism              Balancer                Insurance
```

---

## Use Case 1: DeFi Bridge Security Gates

**Who:** Bridge operators, protocol teams  
**Problem:** Bridge operators need to stop routing when the destination chain or bridge itself becomes too risky.  
**Solution:** Gate transaction acceptance based on Rayachain risk scores.

### How It Works

Polygon (or any bridge operator) integrates Rayachain risk data on-chain via Chainlink oracle:

```solidity
// Bridge gating contract (Polygon side)
pragma solidity ^0.8.22;

import { AggregatorV3Interface } from "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PolygonBridgeGate {
    AggregatorV3Interface public ethereumRisk;    // Risk of Ethereum mainnet
    AggregatorV3Interface public polyBridgeRisk;  // Risk of Polygon bridge itself
    
    uint8 public softLimit = 50;   // Warn users
    uint8 public hardLimit = 75;   // Refuse transfers
    
    function canBridgeToEthereum(uint256 amount) external view returns (bool) {
        (,int256 ethRisk,,,) = ethereumRisk.latestRoundData();
        (,int256 bridgeRisk,,,) = polyBridgeRisk.latestRoundData();
        
        uint8 combinedRisk = uint8((int256(ethRisk) + int256(bridgeRisk)) / 2);
        
        if (combinedRisk >= hardLimit) {
            return false;  // Block all transfers
        }
        if (combinedRisk >= softLimit) {
            emit HighRiskWarning(amount, combinedRisk);  // Warn but allow
        }
        
        return true;
    }
    
    function bridge(
        address token,
        uint256 amount,
        address recipient
    ) external payable {
        require(canBridgeToEthereum(amount), "Risk too high; bridge gated");
        
        // Proceed with bridge (send to underlying bridge contract)
        // ...
    }
}
```

### Outcome

- **Before Rayachain:** Nomad Bridge's $190M loss happened *after* it was deployed. No on-chain gating existed.
- **With Rayachain:** When a bridge's risk score hits 75, new transfers are refused. Existing bridged tokens can still exit.
- **User experience:** Users see "Bridge temporarily gated due to high risk" and wait hours/days instead of losing $190M.

**Adoption examples (potential):**
- Polygon (bridges to Arbitrum, Optimism)
- Arbitrum (bridges to Ethereum, Polygon)
- Optimism (bridges to Ethereum, Arbitrum)

---

## Use Case 2: Cross-Chain Liquidity Protection

**Who:** Market makers, LP protocol developers  
**Problem:** Market makers on Uniswap V4 provide liquidity to bridge settlement pools. If a bridge fails, the pool can be drained. If a chain consensus halts, the pool is locked.  
**Solution:** Uniswap V4 hooks can monitor Rayachain risk and adjust slippage/fees dynamically.

### How It Works

Uniswap V4 Hooks allow protocols to insert custom logic into swaps. A liquidity-protection hook monitors Rayachain:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import { BaseHook } from "v4-periphery/BaseHook.sol";
import { IPoolManager } from "v4-core/IPoolManager.sol";
import { AggregatorV3Interface } from "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract RayachainLiquidityHook is BaseHook {
    IPoolManager public poolManager;
    AggregatorV3Interface public bridgeRiskFeed;
    
    mapping(bytes32 => uint8) public poolRiskThresholds;
    
    /**
     * onBeforeSwap: Adjust swap fee based on Rayachain risk.
     * High risk → higher fee → discourages LPs from exiting pool.
     */
    function beforeSwap(
        address sender,
        IPoolManager.SwapParams calldata params,
        bytes calldata hookData
    ) external override returns (bytes4) {
        (,int256 bridgeRisk,,,) = bridgeRiskFeed.latestRoundData();
        
        uint8 risk = uint8(bridgeRisk);
        uint24 baseFee = 3000;  // 0.3%
        uint24 adjustedFee;
        
        if (risk < 30) {
            adjustedFee = baseFee;  // Normal fee
        } else if (risk < 50) {
            adjustedFee = baseFee + 1000;  // 0.4%
        } else if (risk < 70) {
            adjustedFee = baseFee + 3000;  // 0.6%
        } else {
            adjustedFee = 10000;  // 1% — strong disincentive
        }
        
        // Store adjusted fee (actual update happens in _modifyFee callback)
        // This is pseudocode; real implementation uses Uniswap's fee hook mechanism
        
        return this.beforeSwap.selector;
    }
    
    function afterSwap(
        address sender,
        IPoolManager.SwapParams calldata params,
        IPoolManager.SwapDelta calldata delta,
        bytes calldata hookData
    ) external override returns (bytes4) {
        // Post-swap logging, event emission, etc.
        return this.afterSwap.selector;
    }
}
```

### Outcome

- **Without hook:** Bridge fails → settlement pool drained → LPs lose capital.
- **With hook:** Risk score rises 50–75 → fees increase 0.3% → 0.6%–1% → LPs who want to exit must pay premium → capital is preserved.
- **Market signal:** Higher fees signal to arbitrageurs "this pair is risky" → they reduce position size → pool pressure eases.

**Adoption examples (potential):**
- Uniswap V4 (settlement pool for Nomad, Poly Network, Ronin bridges)
- Curve (stablecoin bridge pairs)
- Balancer (multi-token settlement pools)

---

## Use Case 3: Institutional Custody Risk Monitoring

**Who:** Custody providers (Coinbase, Kraken, Fidelity), asset managers  
**Problem:** Institutional clients hold assets across multiple chains (Ethereum, Polygon, Solana, Cosmos). Custody providers need to know if an asset is "stuck" due to bridge failure.  
**Solution:** Monitor Rayachain risk for all bridges + destination chains; alert clients when it's unsafe to move assets.

### How It Works

A custody provider integrates Rayachain into its risk dashboard:

```python
# Pseudocode: Custody risk dashboard backend
from rayachain_api import RayachainClient

class CustodyRiskMonitor:
    def __init__(self):
        self.client = RayachainClient(api_key="...")
        self.customer_positions = {
            "customer_1": {
                "ethereum": 100,    # 100 ETH on Ethereum
                "polygon": 50,      # 50 ETH on Polygon (bridged via Polygon bridge)
                "arbitrum": 30,     # 30 ETH on Arbitrum (bridged via Stargate/LayerZero)
            }
        }
    
    def check_custody_safety(self, customer_id):
        positions = self.customer_positions[customer_id]
        at_risk = []
        
        for chain, amount in positions.items():
            # Get risk for the bridge that brought assets to this chain
            bridge_id = self.get_bridge_id(chain)
            risk = self.client.get_current_risk(bridge_id)
            
            if risk.score > 60:
                at_risk.append({
                    "chain": chain,
                    "amount": amount,
                    "bridge": bridge_id,
                    "risk_score": risk.score,
                    "action": "WAIT" if risk.score < 75 else "EMERGENCY_EXIT"
                })
        
        if at_risk:
            self.notify_customer(customer_id, at_risk)
        
        return at_risk
    
    def notify_customer(self, customer_id, at_risk_positions):
        """Send alert to customer + their compliance team"""
        # Slack/email/dashboard update
        print(f"[{customer_id}] Custody risk alert:")
        for pos in at_risk_positions:
            print(f"  {pos['amount']} {pos['chain']} has bridge risk {pos['risk_score']}/100")
```

### Outcome

- **Without monitoring:** Bridge fails silently → customer's assets stuck on risky chain → reputation damage.
- **With monitoring:** Risk spike detected → customer is alerted proactively → can move assets before failure → trust maintained.

**Adoption examples (potential):**
- Coinbase Institutional (custody of institutional crypto)
- Fidelity Digital Assets (high-net-worth custody)
- Kraken Futures (collateral management)

---

## Use Case 4: Validator Incentive Alignment

**Who:** Bittensor subnet validators, node operators  
**Problem:** Validators need to be paid for their work. Rayachain generates value (risk data) that others consume. How do validators capture that value?  
**Solution:** Validators earn TAO emissions for accurate risk signals; TAO is staked into Rayachain subnet; subnet receives Taoflow emission share.

### How It Works

1. **Validator provides signal:** "Bridge risk = 45" (based on monitoring finality, config, liquidity, etc.)
2. **Other validators vote:** 80% agree the signal is accurate.
3. **Accuracy is recorded:** This validator's signal was correct.
4. **Rewards are distributed:** Next epoch, this validator's weight increases.
5. **Emissions increase:** More TAO flows into subnet → validator earns more.

**Revenue model:**

```
Risk data consumers (protocols, custodians, MMs)
                   │
                   │ Pay $USDC for API access
                   ▼
            Rayachain DAO
                   │
                   │ Buy TAO with USDC fees
                   │ Stake into Rayachain subnet
                   ▼
           Bittensor Consensus
                   │
                   │ Emit TAO to subnet participants
                   ▼
        ┌──────────┼──────────┐
        │          │          │
      Miners   Validators  DAO Treasury
      (41%)     (41%)       (18%)
```

### Outcome

- **Without incentive alignment:** Validators are volunteers → quality degrades → risk signals become unreliable.
- **With incentive alignment:** Validators are paid in TAO → quality improves → risk signals are trusted → adoption increases.

**Participation examples (potential):**
- Risk researchers earn TAO for running validator nodes.
- DeFi protocols earn TAO by validating their own bridge risk.
- Insurance companies earn TAO for accurate loss predictions.

---

## Integration Timeline

### Phase 1: Early Adopters (Q2 2026)
- Protocol teams integrate via Chainlink oracle (testnet).
- Market makers experiment with Uniswap V4 hooks (local testing).
- Custody providers integrate risk API into dashboards (manual alerts).

### Phase 2: Mainnet Launch (Q3 2026)
- Risk oracle deployed on Ethereum, Arbitrum, Polygon, Optimism, Base.
- 50+ validators running on Bittensor subnet.
- 3–5 protocol teams using bridge gates in production.

### Phase 3: Scaled Adoption (Q4 2026+)
- 200+ validators.
- 20+ protocols gating bridges based on Rayachain.
- Rayachain becomes the standard risk feed for cross-chain DeFi.

---

## Getting Started

1. **Protocols:** [API Examples](04-api-examples.md) — integrate risk gating.
2. **Market makers:** [Architecture](02-architecture.md) — understand the data sources.
3. **Validators:** [Bittensor Docs](https://docs.learnbittensor.org/) — run a validator node.
4. **Custody providers:** Contact [LINK_NEEDED: sales@rayachain.io] for enterprise setup.

---

**Learn More:**
- [OmniRisk.io](https://omnirisk.io) — Risk intelligence for crypto
- [Rayachain.omnirisk.io](https://rayachain.omnirisk.io/) — Cross-chain technology

**Questions?** Open an issue on [GitHub](github.com/baz2024/rayachain-oft) or ask in [Discord](LINK_NEEDED).
