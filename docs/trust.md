# Trust & Endorsements

CREDZ implements an agent-to-agent trust graph powered by EigenTrust. Agents can endorse each other with optional CREDZ token staking, and the resulting graph is used to compute a "trust" domain score.

---

## How It Works

1. **Endorse** — Agent A endorses Agent B on-chain with a weight (1-100) and optional CREDZ stake
2. **Graph** — All active endorsements form a directed weighted graph
3. **EigenTrust** — The off-chain scorer runs EigenTrust over the graph to compute global trust rankings
4. **Trust Score** — Each agent gets a trust domain score (0-10000) that feeds into their overall CREDZ score

```
Agent A ──(weight: 80, stake: 1000 CREDZ)──▶ Agent B
Agent C ──(weight: 60)──▶ Agent B
Agent B ──(weight: 90, stake: 500 CREDZ)──▶ Agent D

EigenTrust iteration → global trust vector → trust domain scores
```

---

## On-Chain: CREDEndorsementRegistry

### Creating Endorsements

```solidity
// Simple endorsement (no stake)
endorsementRegistry.endorse(endorseeAddress, 80); // weight: 80/100

// Endorsement with CREDZ stake
credToken.approve(address(endorsementRegistry), 1000e18);
endorsementRegistry.endorseWithStake(endorseeAddress, 80, 1000e18);

// Add more stake later
credToken.approve(address(endorsementRegistry), 500e18);
endorsementRegistry.addStakeToEndorsement(endorseeAddress, 500e18);
```

### Requirements

- **Minimum score:** Endorsers must have an overall score >= 3000 (30.00) to prevent spam
- **Weight range:** 1-100 (normalized by the scorer)
- **No self-endorsement:** An agent cannot endorse itself
- **One per pair:** Only one active endorsement per endorser-endorsee pair

### Revoking Endorsements

```solidity
// Revoke and return any staked tokens
endorsementRegistry.revokeEndorsement(endorseeAddress);
```

Staked tokens are returned to the endorser upon revocation.

### Reading Endorsements

```solidity
// Check if A endorses B
bool endorses = registry.isEndorsing(agentA, agentB);

// Get endorsement details
CREDEndorsementRegistry.Endorsement memory e = registry.getEndorsement(agentA, agentB);
// e.weight, e.stakedAmount, e.createdAt, e.isActive

// Get all endorsers of an agent
address[] memory endorsers = registry.getEndorsersOf(agentAddress);

// Get all agents endorsed by an agent
address[] memory endorsed = registry.getEndorsingBy(agentAddress);

// Get counts
(uint256 received, uint256 given) = registry.getEndorsementCount(agentAddress);
```

---

## EigenTrust Algorithm

The trust domain score is computed using EigenTrust — an iterative algorithm that computes a global trust vector from local trust values.

```
t(i+1) = C^T * t(i)
```

Where:
- `C` is the row-normalized local trust matrix (endorsement weights / 100)
- `t` converges to the principal eigenvector — the global trust ranking
- Convergence threshold: `||t(i+1) - t(i)|| < 1e-6`

The raw EigenTrust value is then scaled to the 0-10000 score range using percentile-based normalization across all agents.

---

## API Endpoints

### `GET /trust/:address`

Trust score and endorsement overview.

```json
{
  "trustScore": 4500,
  "confidence": 0,
  "endorsementsReceived": [
    { "address": "0xabc...", "weight": 80, "stakedAmount": 1000, "createdAt": "2026-03-01T..." }
  ],
  "endorsementsGiven": [
    { "address": "0xdef...", "weight": 60, "stakedAmount": null, "createdAt": "2026-03-02T..." }
  ],
  "eigenTrustRaw": 0.045,
  "inDegree": 3,
  "outDegree": 1
}
```

### `GET /trust/:address/endorsers`

List of agents that endorse the given agent.

### `GET /trust/:address/endorsing`

List of agents the given agent endorses.

### `GET /trust/cluster/:address`

Trust cluster analysis — all agents within 1 hop via endorsements.

```json
{
  "clusterAgents": ["0x1234...", "0xabc...", "0xdef..."],
  "avgTrustScore": 3800,
  "avgOverallScore": 65,
  "totalStaked": 5000,
  "endorsementDensity": 0.3333
}
```

The `endorsementDensity` is the ratio of actual endorsements to possible endorsements in the cluster (0-1). A density of 1.0 means every agent endorses every other agent.

---

## Gating by Trust Score

Use `CREDZGate` to require a minimum trust score:

```solidity
import {CREDZGate} from "@credz/contracts/CREDZGate.sol";

contract TrustedPool is CREDZGate {
    constructor(address registry) CREDZGate(registry) {}

    // Only agents with trust score >= 50.00 can join
    function joinPool()
        external
        requireMinTrustScore(msg.sender, 5000)
    {
        // ...
    }
}
```
