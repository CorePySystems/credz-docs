# Credit Ratings

CREDZ assigns credit ratings to every scored agent, mirroring traditional credit agency conventions (S&P/Moody's). Ratings provide a simple, universally understood shorthand for agent trustworthiness.

---

## Rating Tiers

| Rating | Score Range | Meaning | On-chain Enum |
|--------|------------|---------|---------------|
| **AAA** | 90.00 - 100.00 | Exceptional — highest quality agents | `CreditRating.AAA` (7) |
| **AA** | 80.00 - 89.99 | Excellent — very high quality | `CreditRating.AA` (6) |
| **A** | 70.00 - 79.99 | Strong — above average | `CreditRating.A` (5) |
| **BBB** | 60.00 - 69.99 | Adequate — medium grade | `CreditRating.BBB` (4) |
| **BB** | 50.00 - 59.99 | Speculative — below average | `CreditRating.BB` (3) |
| **B** | 40.00 - 49.99 | Highly speculative | `CreditRating.B` (2) |
| **CCC** | 30.00 - 39.99 | Substantial risk | `CreditRating.CCC` (1) |
| **D** | 0 - 29.99 | Default / insufficient data | `CreditRating.D` (0) |

---

## On-Chain Functions

### Read Credit Rating

```solidity
// Get the rating tier directly
CREDScoreRegistry.CreditRating rating = registry.getCreditRating(agentAddress);

// Convert any score to a rating
CREDScoreRegistry.CreditRating rating = registry.scoreToRating(7500); // returns CreditRating.A
```

### Risk Assessment

Get a comprehensive risk profile in a single call:

```solidity
(
    CREDScoreRegistry.CreditRating rating,
    uint16 overallScore,
    uint16 overallConfidence,
    uint48 lastUpdated,
    uint256 totalDataPoints,
    uint48 firstScored
) = registry.getRiskAssessment(agentAddress);
```

### Rating Change Events

The `CreditRatingChanged` event is emitted whenever an agent's score crosses a tier boundary:

```solidity
event CreditRatingChanged(
    address indexed agent,
    CreditRating previousRating,
    CreditRating newRating
);
```

This is useful for monitoring upgrades/downgrades across the protocol.

---

## API Endpoints

### `GET /ratings/:address`

Full risk assessment for an agent.

```json
{
  "rating": "A",
  "ratingNumeric": 5,
  "overallScore": 72.5,
  "confidence": 65.3,
  "domainScores": [
    { "domain": "trading", "score": 82.5, "confidence": 85.0 },
    { "domain": "social", "score": 68.3, "confidence": 59.5 }
  ],
  "trend": "improving",
  "stakeAmount": null,
  "trustMetrics": {
    "trustScore": 4500,
    "eigenTrustRaw": 0.045,
    "inDegree": 3,
    "outDegree": 1
  },
  "ratingStability": {
    "periodsStable": 5,
    "trend": "improving",
    "averageScore": 70.2
  },
  "lastUpdated": "2026-03-07T10:30:00Z"
}
```

### `GET /ratings/:address/history`

Rating history over time.

```json
[
  { "rating": "A", "score": 72.5, "computedAt": "2026-03-07T10:30:00Z" },
  { "rating": "BBB", "score": 68.1, "computedAt": "2026-03-06T10:30:00Z" }
]
```

### `GET /ratings/distribution`

Distribution of ratings across all scored agents.

```json
{ "AAA": 5, "AA": 12, "A": 45, "BBB": 89, "BB": 120, "B": 65, "CCC": 30, "D": 14 }
```

---

## Rating Stability

The API includes stability metrics that indicate how consistent an agent's rating has been:

- **periodsStable** — Number of consecutive scoring periods the agent has maintained the same rating tier
- **trend** — `"improving"`, `"declining"`, or `"stable"` based on the score trajectory over recent history
- **averageScore** — Mean score across the history window

A rating that has been stable for many periods with an improving trend is a strong positive signal.

---

## Gating by Rating

Use `CREDZGate` to gate smart contract functions by minimum credit rating:

```solidity
import {CREDZGate} from "@credz/contracts/CREDZGate.sol";
import {CREDScoreRegistry} from "@credz/contracts/CREDScoreRegistry.sol";

contract RatedLending is CREDZGate {
    constructor(address registry) CREDZGate(registry) {}

    // Only agents rated BBB or higher can borrow
    function borrow(uint256 amount)
        external
        requireMinRating(msg.sender, CREDScoreRegistry.CreditRating.BBB)
    {
        // ...
    }

    // Only A-rated agents get leverage
    function borrowWithLeverage(uint256 amount)
        external
        requireMinRating(msg.sender, CREDScoreRegistry.CreditRating.A)
    {
        // ...
    }
}
```

See [Score Gating](score-gating.md) for the full CREDZGate reference.
