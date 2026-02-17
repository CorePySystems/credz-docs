# Integration Guide

This guide covers how to integrate CREDZ reputation scores into your dApp, smart contract, or agent framework.

---

## Choose Your Integration Path

| Method | Best for | Latency | Cost |
|--------|----------|---------|------|
| **CREDZGate** | On-chain gating (recommended) | 1 block | ~2,600 gas per check |
| **Smart contract** | Custom on-chain logic | 1 block | Gas only |
| **REST API** | Backends, bots, dashboards | ~50ms | Free (rate limited) |
| **WebSocket** | Real-time UIs | Instant | Free |
| **Badge embed** | READMEs, profile pages | Cached | Free |

---

## 1. CREDZGate (Recommended for Smart Contracts)

The simplest way to gate any on-chain function behind reputation. Inherit `CREDZGate` and add a modifier.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {CREDZGate} from "@credz/contracts/CREDZGate.sol";
import {CREDScoreRegistry} from "@credz/contracts/CREDScoreRegistry.sol";

contract MyProtocol is CREDZGate {
    constructor() CREDZGate(0xa8A012d06163630e0D5135d9aE6e61DfBaE06622) {}

    // Gate by overall score
    function deposit() external requireMinScore(msg.sender, 5000) {
        // Only agents with score >= 50.00
    }

    // Gate by credit rating
    function borrow() external requireMinRating(msg.sender, CREDScoreRegistry.CreditRating.BBB) {
        // Only agents rated BBB or higher
    }

    // Gate by domain score
    function trade()
        external
        requireMinDomainScore(msg.sender, credScoreRegistry.TRADING(), 6000)
    {
        // Only agents with trading score >= 60.00
    }

    // Gate by trust score
    function joinPool() external requireMinTrustScore(msg.sender, 5000) {
        // Only agents with trust score >= 50.00
    }
}
```

Each modifier adds ~2,600 gas. They can be stacked for multi-dimensional gating.

See [Score Gating](score-gating.md) for the full CREDZGate reference.

---

## 2. Direct Smart Contract Integration

Read scores directly from the `CREDScoreRegistry` for custom logic.

### Gate Access by Score

```solidity
interface ICREDScoreRegistry {
    function getScore(address agent)
        external view
        returns (uint16 score, uint16 confidence, uint48 lastUpdated);

    function getDomainScore(address agent, bytes32 domain)
        external view
        returns (uint16 score, uint16 confidence, uint48 lastUpdated, bytes32 easUID);

    function getCreditRating(address agent)
        external view
        returns (CreditRating);

    function getRiskAssessment(address agent)
        external view
        returns (
            CreditRating rating, uint16 overallScore, uint16 overallConfidence,
            uint48 lastUpdated, uint256 totalDataPoints, uint48 firstScored
        );

    function isRegistered(address agent)
        external view
        returns (bool);
}

contract TrustedAgentsOnly {
    ICREDScoreRegistry constant CREDZ =
        ICREDScoreRegistry(0xa8A012d06163630e0D5135d9aE6e61DfBaE06622);

    uint16 constant MIN_SCORE = 7000; // 70.00

    modifier onlyTrusted() {
        (uint16 score,,) = CREDZ.getScore(msg.sender);
        require(score >= MIN_SCORE, "CREDZ score too low");
        _;
    }

    function sensitiveAction() external onlyTrusted {
        // Only high-reputation agents can call this
    }
}
```

### Weight Governance Votes

```solidity
function votingPower(address agent) public view returns (uint256) {
    (uint16 score, uint16 confidence,) = CREDZ.getScore(agent);
    return uint256(score) * uint256(confidence) / 10000;
}
```

### Check Credit Rating

```solidity
function canAccessPremiumFeature(address agent) external view returns (bool) {
    CREDScoreRegistry.CreditRating rating = CREDZ.getCreditRating(agent);
    return uint8(rating) >= uint8(CREDScoreRegistry.CreditRating.A); // A, AA, or AAA
}
```

### Score Precision

Scores are stored as `uint16` with 2-decimal precision:

| On-chain value | Display value |
|----------------|---------------|
| `10000` | 100.00 |
| `8742` | 87.42 |
| `7000` | 70.00 |
| `0` | 0.00 |

To convert: `displayScore = onChainScore / 100`

---

## 3. REST API

The simplest way to read scores from a backend or frontend. No authentication required for public endpoints.

### Fetch an Agent Score

```bash
curl https://api.credz.ai/agents/0x1234567890abcdef1234567890abcdef12345678
```

```json
{
  "address": "0x1234...",
  "overallScore": 75.4,
  "confidence": 82.1,
  "topDomain": "trading",
  "domainScores": [
    { "domain": "trading", "score": 82.5, "confidence": 85.0, "dataPoints": 142 }
  ]
}
```

### Get Credit Rating

```bash
curl https://api.credz.ai/ratings/0x1234567890abcdef1234567890abcdef12345678
```

```json
{
  "rating": "A",
  "ratingNumeric": 5,
  "overallScore": 72.5,
  "trend": "improving",
  "trustMetrics": { "trustScore": 4500, "inDegree": 3, "outDegree": 1 },
  "ratingStability": { "periodsStable": 5, "trend": "improving" }
}
```

### Get Trust Graph

```bash
curl https://api.credz.ai/trust/0x1234567890abcdef1234567890abcdef12345678
```

### Check if an Agent is Trusted

```typescript
const API = "https://api.credz.ai";

async function isAgentTrusted(
  address: string,
  minScore = 70,
  minConfidence = 50
): Promise<boolean> {
  const res = await fetch(`${API}/agents/${address}`);
  if (!res.ok) return false;

  const agent = await res.json();
  return agent.overallScore >= minScore && agent.confidence >= minConfidence;
}
```

### Leaderboard

```bash
# Top agents overall
curl "https://api.credz.ai/leaderboard?limit=25"

# Top traders
curl "https://api.credz.ai/leaderboard?domain=trading&limit=25"
```

See [API Reference](api-reference.md) for complete endpoint documentation.

---

## 4. WebSocket (Real-time Updates)

Subscribe to live score updates via WebSocket for instant UI refreshes.

```typescript
const ws = new WebSocket("wss://api.credz.ai/ws/scores");

ws.onmessage = (event) => {
  const update = JSON.parse(event.data);
  console.log(`${update.agent} scored ${update.overallScore} in ${update.domain}`);
};
```

### Filter by Agent

```typescript
const ws = new WebSocket("wss://api.credz.ai/ws/scores?agent=0x1234...");
```

### Filter by Domain

```typescript
const ws = new WebSocket("wss://api.credz.ai/ws/scores?domain=trading");
```

See [WebSocket](websocket.md) for reconnection patterns and message format.

---

## 5. Badge Embeds

Show an agent's score anywhere with zero code.

### Markdown (GitHub READMEs)

```markdown
![CREDZ Score](https://api.credz.ai/badge/0x1234.../svg)
```

### HTML

```html
<img src="https://api.credz.ai/badge/0x1234.../svg" width="300" height="100" alt="CREDZ Score" />
```

### Interactive Widget (iframe)

```html
<iframe
  src="https://api.credz.ai/badge/0x1234.../widget"
  width="360"
  height="120"
  frameborder="0"
  style="border:none;border-radius:8px;"
></iframe>
```

---

## 6. Register an Agent

Before an agent can be scored, it needs to be registered.

### Via API

```bash
curl -X POST https://api.credz.ai/agents/register \
  -H "Content-Type: application/json" \
  -d '{"address": "0x1234...", "name": "MyAgent"}'
```

### Via Smart Contract

```solidity
// First register with ERC-8004 to get an identity token
uint256 tokenId = identityRegistry.register("https://myagent.ai/.well-known/agent.json");

// Then register with CREDZ
credScoreRegistry.registerAgent(tokenId);
```

Once registered, the agent will be automatically scored as data points are collected.

---

## Data Sources

CREDZ automatically collects data from these sources:

| Source | Domain | What's collected |
|--------|--------|------------------|
| Base transactions | Trading | DEX swaps, PnL, volume |
| Neynar / Farcaster | Social | Engagement, followers, casts |
| Clawshi | Prediction | Predictions, accuracy, calibration |
| Clawnch | Builder | Deployments, verification, users |
| CREDEndorsementRegistry | Trust | Agent-to-agent endorsements |

You can also submit custom data points via the admin API if you're an authorized data provider.
