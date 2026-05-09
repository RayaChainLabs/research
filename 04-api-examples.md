# Rayachain API: Real-World Code Examples

**For:** Developers, integrators, protocol builders  
**Duration:** 10–20 minutes  
**Complements:** [Architecture](02-architecture.md) | [Getting Started](01-getting-started.md)

---

## What Can You Query?

The Rayachain API exposes two main capability sets:

### 1. Token Transfer Endpoints

- **GET /v1/rayachain/dashboard** — Operator dashboard with transfer metrics
- **GET /v1/rayachain/transfers** — List transfers with filtering and pagination
- **GET /v1/rayachain/transfers/:transferId** — Full transfer details
- **POST /v1/rayachain/transfers/quote** — Get LayerZero fee quote
- **POST /v1/rayachain/transfers/build** — Build a signed transfer transaction

### 2. Public Risk Broadcast Endpoints

- **POST /v1/rayachain/broadcast/plan** — Plan a broadcast to one or more chains
- **POST /v1/rayachain/broadcast/quote** — Get fee quote for broadcast
- **POST /v1/rayachain/broadcast/send** — Submit a broadcast
- **GET /v1/rayachain/broadcast/:id** — Track broadcast delivery
- **GET /v1/rayachain/broadcast/:id/destination/:chain** — Per-destination status

### 3. Infrastructure Endpoints

- **GET /v1/rayachain/chains** — Supported chain registry
- **GET /v1/rayachain/routes** — Available transfer/broadcast routes
- **GET /v1/rayachain/routes/health** — Per-route health status

All endpoints follow a standard response envelope with `data`, `meta` (request ID, timestamp, version), and optional `error` fields. See [API Documentation][LINK_NEEDED] for full details.

---

## Authentication

### Create an API Key

1. Sign in to [Rayachain Dashboard][LINK_NEEDED]
2. Navigate to **Settings → API Keys**
3. Click **Create New Key**
4. Copy your key and store it securely (never commit to git)

### Using the API Key

All requests require the `Authorization` header:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  https://api.rayachain.io/v1/risk/current
```

In your code:

```python
# Python + requests
import requests

RAYACHAIN_API_KEY = "rya_live_abc123..."

headers = {
    "Authorization": f"Bearer {RAYACHAIN_API_KEY}"
}

response = requests.get(
    "https://api.rayachain.io/v1/risk/current",
    headers=headers
)
```

```javascript
// JavaScript + fetch
const RAYACHAIN_API_KEY = "rya_live_abc123...";

fetch("https://api.rayachain.io/v1/risk/current", {
    headers: {
        "Authorization": `Bearer ${RAYACHAIN_API_KEY}`
    }
});
```

---

## Example 1: Query Current Risk for a Bridge

Get the latest risk score for a specific bridge.

**Request:**

```bash
curl -H "Authorization: Bearer $RAYACHAIN_API_KEY" \
  "https://api.rayachain.io/v1/risk/current?bridge_id=nomad_eth_moon"
```

**Response:**

```json
{
  "bridge_id": "nomad_eth_moon",
  "risk_score": 47,
  "risk_category": "medium",
  "confidence": 0.92,
  "last_updated": "2026-05-09T16:00:00Z",
  "components": {
    "finality_risk": 10,
    "config_risk": 35,
    "liquidity_risk": 60,
    "operational_risk": 20,
    "economic_risk": 45
  },
  "recent_anomalies": [
    {
      "timestamp": "2026-05-09T15:45:00Z",
      "type": "liquidity_outflow",
      "severity": "moderate",
      "description": "2.3M USDC withdrawn from settlement pool in 8 minutes"
    }
  ]
}
```

**Interpretation:**

- **risk_score: 47** — Medium risk, not urgent but worth monitoring.
- **confidence: 0.92** — 92% of validators agreed on this score (high consensus).
- **liquidity_risk: 60** — The main concern is settlement pool health, not the bridge code itself.

---

## Example 2: Fetch Historical Risk Data

Get risk scores over a time range for trending and alerting.

**Request:**

```bash
curl -H "Authorization: Bearer $RAYACHAIN_API_KEY" \
  "https://api.rayachain.io/v1/risk/history?bridge_id=lz_arb_base&from=2026-05-07T00:00:00Z&to=2026-05-09T23:59:59Z&granularity=1h"
```

**Response:**

```json
{
  "bridge_id": "lz_arb_base",
  "data": [
    {
      "timestamp": "2026-05-07T00:00:00Z",
      "risk_score": 25,
      "confidence": 0.85
    },
    {
      "timestamp": "2026-05-07T01:00:00Z",
      "risk_score": 26,
      "confidence": 0.86
    },
    // ... more hourly entries
    {
      "timestamp": "2026-05-09T23:00:00Z",
      "risk_score": 22,
      "confidence": 0.91
    }
  ],
  "min_score": 20,
  "max_score": 34,
  "avg_score": 25.3,
  "trend": "stable"
}
```

**Use case:** Integrate this into your dashboard to show users how a bridge's risk has evolved over days/weeks.

---

## Example 3: Subscribe to Risk Alerts via Webhook

Get notified immediately when risk spikes above a threshold.

**1. Set up a webhook endpoint in your app:**

```python
# Flask example
from flask import Flask, request

app = Flask(__name__)

@app.route("/webhooks/rayachain", methods=["POST"])
def handle_rayachain_alert():
    payload = request.json
    
    # payload contains:
    # {
    #   "event": "risk_alert",
    #   "bridge_id": "nomad_eth_moon",
    #   "previous_score": 35,
    #   "current_score": 72,
    #   "threshold": 60,
    #   "timestamp": "2026-05-09T16:15:00Z",
    #   "anomalies": [...]
    # }
    
    if payload["current_score"] > 70:
        send_urgent_alert(payload)  # Your alert logic
    else:
        log_risk_update(payload)
    
    return {"status": "ok"}, 200
```

**2. Register the webhook in the API:**

```bash
curl -X POST \
  -H "Authorization: Bearer $RAYACHAIN_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "event": "risk_threshold_exceeded",
    "bridge_id": "nomad_eth_moon",
    "threshold": 60,
    "webhook_url": "https://yourapp.com/webhooks/rayachain"
  }' \
  https://api.rayachain.io/v1/alerts/subscribe
```

**3. Verify the webhook:**

Rayachain will send a test event to your webhook. Respond with `{"ok": true}` to confirm.

---

## Example 4: Integrate Risk Scores Into a Smart Contract

Use Chainlink or Pyth to bring Rayachain risk data on-chain.

**Pseudocode:**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import { AggregatorV3Interface } from "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract BridgeRiskGate {
    AggregatorV3Interface public riskFeed;
    uint8 public maxRiskThreshold = 60;
    
    constructor(address riskFeedAddress) {
        riskFeed = AggregatorV3Interface(riskFeedAddress);
    }
    
    function canBridge(
        address token,
        uint256 amount,
        uint32 destinationChain
    ) public view returns (bool) {
        (,int256 riskScore,,,) = riskFeed.latestRoundData();
        
        // Only allow bridge if risk is below threshold
        return uint8(riskScore) < maxRiskThreshold;
    }
    
    function bridgeToken(
        address token,
        uint256 amount,
        uint32 destinationChain
    ) external {
        require(canBridge(token, amount, destinationChain), "Risk too high");
        
        // Proceed with bridge...
        // (OFT.send call, etc.)
    }
}
```

**How it works:**

1. Chainlink oracle pushes Rayachain risk data on-chain every 12 minutes.
2. Your smart contract queries the latest risk via `latestRoundData()`.
3. If risk > threshold, bridging is gated.

---

## Example 5: Webhook Receiver (Node.js)

A complete Node.js webhook server for receiving Rayachain alerts.

```javascript
import express from "express";
import crypto from "crypto";

const app = express();
app.use(express.json());

const RAYACHAIN_WEBHOOK_SECRET = "whsec_abc123..."; // From Rayachain Dashboard

// Verify webhook signature
function verifyWebhookSignature(req) {
    const signature = req.headers["x-rayachain-signature"];
    const timestamp = req.headers["x-rayachain-timestamp"];
    const body = JSON.stringify(req.body);
    
    const payload = `${timestamp}.${body}`;
    const expectedSignature = crypto
        .createHmac("sha256", RAYACHAIN_WEBHOOK_SECRET)
        .update(payload)
        .digest("hex");
    
    return signature === expectedSignature;
}

app.post("/alerts/rayachain", (req, res) => {
    // Verify webhook is genuine (prevents replay attacks)
    if (!verifyWebhookSignature(req)) {
        return res.status(403).json({ error: "Invalid signature" });
    }
    
    const { event, bridge_id, current_score, previous_score, anomalies } = req.body;
    
    console.log(`[${event}] Bridge ${bridge_id}: ${previous_score} → ${current_score}`);
    
    if (anomalies && anomalies.length > 0) {
        console.log("Anomalies detected:");
        anomalies.forEach(a => {
            console.log(`  - ${a.type}: ${a.description}`);
        });
    }
    
    // Your alert logic here
    if (current_score > 70) {
        sendToSlack(`🚨 High-risk bridge: ${bridge_id} (${current_score}/100)`);
    }
    
    res.json({ status: "ok" });
});

app.listen(3000, () => {
    console.log("Webhook server running on :3000");
});
```

---

## Error Handling

### Common Errors

#### 401 Unauthorized

Your API key is missing or invalid.

```json
{
  "error": "Unauthorized",
  "message": "Invalid API key"
}
```

**Fix:** Check your key in the Dashboard; rotate if necessary.

#### 429 Too Many Requests

You've exceeded your rate limit.

```json
{
  "error": "RateLimitExceeded",
  "message": "100 requests per minute",
  "retry_after_seconds": 60
}
```

**Fix:** Implement exponential backoff or upgrade your plan.

#### 404 Not Found

The bridge ID doesn't exist.

```json
{
  "error": "NotFound",
  "message": "Bridge nomad_eth_moon not found"
}
```

**Fix:** Check the [Bridge Registry][LINK_NEEDED] for valid IDs.

### Retry Logic

Implement exponential backoff for transient failures:

```python
import time
import requests

def fetch_with_retry(url, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.get(
                url,
                headers={"Authorization": f"Bearer {RAYACHAIN_API_KEY}"},
                timeout=10
            )
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # 1s, 2s, 4s
                print(f"Retry {attempt + 1} in {wait_time}s...")
                time.sleep(wait_time)
            else:
                raise e
```

---

## Rate Limits & Billing

[LINK_NEEDED: Detailed rate limit table and pricing]

| Plan | Requests/min | Webhooks | Price |
|------|--------------|----------|-------|
| Free | 10 | 1 | $0 |
| Pro | 1,000 | 10 | $99/mo |
| Enterprise | Unlimited | Unlimited | Custom |

---

## Support

- **Discord:** [LINK_NEEDED]
- **Email:** support@rayachain.io
- **GitHub Issues:** [github.com/baz2024/rayachain-oft/issues](github.com/baz2024/rayachain-oft/issues)

---

**Ready to integrate?** Start with [Getting Started](01-getting-started.md), then come back here for API details.

**Platform Links:**
- [OmniRisk](https://omnirisk.io) — Risk intelligence platform
- [Rayachain](https://rayachain.omnirisk.io/) — Cross-chain services
