# Score Gating (CREDZGate)

CREDZGate is an abstract contract that any protocol can inherit to gate smart contract functions behind CREDZ reputation requirements. It provides four modifiers covering overall score, domain score, credit rating, and trust score checks.

---

## Why CREDZGate?

- **One line of code** — Add a modifier to any function to gate it by reputation
- **Minimal gas overhead** — Each modifier is one STATICCALL + comparison (~2,600 gas)
- **Zero storage cost** — The registry reference is `immutable` (no SLOAD)
- **Rich error context** — Custom errors include the agent address, actual score, and required threshold

---

## Quick Start

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {CREDZGate} from "@credz/contracts/CREDZGate.sol";
import {CREDScoreRegistry} from "@credz/contracts/CREDScoreRegistry.sol";

contract MyProtocol is CREDZGate {
    constructor() CREDZGate(0xa8A012d06163630e0D5135d9aE6e61DfBaE06622) {}

    function deposit() external requireMinScore(msg.sender, 5000) {
        // Only agents with score >= 50.00
    }
}
```

---

## Available Modifiers

### `requireMinScore`

Gates on the agent's overall score (0-10000).

```solidity
function action() external requireMinScore(msg.sender, 7000) {
    // Score >= 70.00 required
}
```

Reverts with: `ScoreBelowMinimum(agent, actualScore, requiredScore)`

### `requireMinDomainScore`

Gates on a specific domain score.

```solidity
bytes32 constant TRADING = keccak256("trading");

function trade() external requireMinDomainScore(msg.sender, TRADING, 6000) {
    // Trading score >= 60.00 required
}
```

Reverts with: `DomainScoreBelowMinimum(agent, domain, actualScore, requiredScore)`

### `requireMinRating`

Gates on credit rating tier (enum comparison).

```solidity
function borrow()
    external
    requireMinRating(msg.sender, CREDScoreRegistry.CreditRating.BBB)
{
    // BBB or higher required (BBB, A, AA, AAA)
}
```

Reverts with: `RatingBelowMinimum(agent, actualRating, requiredRating)`

### `requireMinTrustScore`

Gates on the trust domain score (EigenTrust-derived).

```solidity
function joinWhitelist() external requireMinTrustScore(msg.sender, 5000) {
    // Trust score >= 50.00 required
}
```

Reverts with: `TrustScoreBelowMinimum(agent, actualScore, requiredScore)`

---

## Stacking Modifiers

Modifiers can be combined for multi-dimensional gating:

```solidity
function leveragedBorrow()
    external
    requireMinRating(msg.sender, CREDScoreRegistry.CreditRating.A)
    requireMinDomainScore(msg.sender, TRADING, 7000)
    requireMinTrustScore(msg.sender, 5000)
{
    // Requires: A rating + trading >= 70 + trust >= 50
}
```

Each modifier adds ~2,600 gas. Three stacked modifiers add ~7,800 gas total.

---

## Full Example: Gated Lending

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {CREDZGate} from "@credz/contracts/CREDZGate.sol";
import {CREDScoreRegistry} from "@credz/contracts/CREDScoreRegistry.sol";

contract CREDZGatedLending is CREDZGate {
    bytes32 constant TRADING = keccak256("trading");

    constructor(address registry) CREDZGate(registry) {}

    /// @notice Standard borrow — requires BBB rating (score >= 60)
    function borrow(uint256 amount)
        external
        requireMinScore(msg.sender, 6000)
    {
        // Process borrow...
    }

    /// @notice Leveraged borrow — requires A rating + strong trading score
    function borrowWithLeverage(uint256 amount)
        external
        requireMinRating(msg.sender, CREDScoreRegistry.CreditRating.A)
        requireMinDomainScore(msg.sender, TRADING, 7000)
    {
        // Process leveraged borrow...
    }

    /// @notice Premium pool access — requires high trust score
    function accessPremiumPool()
        external
        requireMinTrustScore(msg.sender, 5000)
    {
        // Grant access...
    }
}
```

---

## Error Handling

All CREDZGate errors are custom Solidity errors with full context:

```solidity
error ScoreBelowMinimum(address agent, uint16 score, uint16 required);
error DomainScoreBelowMinimum(address agent, bytes32 domain, uint16 score, uint16 required);
error RatingBelowMinimum(address agent, CREDScoreRegistry.CreditRating rating, CREDScoreRegistry.CreditRating required);
error TrustScoreBelowMinimum(address agent, uint16 score, uint16 required);
```

These can be caught and decoded by frontends using viem or ethers:

```typescript
try {
  await contract.borrow(amount);
} catch (error) {
  if (error.name === "ScoreBelowMinimum") {
    const { agent, score, required } = error.args;
    console.log(`Score ${score / 100} is below required ${required / 100}`);
  }
}
```

---

## Deployment

CREDZGate is an abstract contract — it's inherited, not deployed separately. Your contract's constructor passes the CREDScoreRegistry address:

```solidity
constructor() CREDZGate(SCORE_REGISTRY_ADDRESS) {}
```

The registry address is stored as `immutable`, so there's no ongoing storage cost.
