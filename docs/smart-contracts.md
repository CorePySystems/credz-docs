# Smart Contracts

Core contracts are deployed on **Base Mainnet** (chain ID `8453`). Token-dependent contracts (staking, endorsements, query fees) will be deployed after the CREDZ token launch.

---

## Deployed Addresses

| Contract | Address | Description |
|----------|---------|-------------|
| **CREDScoreRegistry** | [`0xa8A012d06163630e0D5135d9aE6e61DfBaE06622`](https://basescan.org/address/0xa8A012d06163630e0D5135d9aE6e61DfBaE06622) | Core scoring registry + credit ratings |
| **DataProviderRegistry** | [`0xaFBd2edc5076EFaf81F5ae456D8c708cDCDCc655`](https://basescan.org/address/0xaFBd2edc5076EFaf81F5ae456D8c708cDCDCc655) | Authorized data providers |
| **ERC8004IdentityStub** | [`0x1af6A661E6B09F203F4e6cb8D13EF1fFEc1a945b`](https://basescan.org/address/0x1af6A661E6B09F203F4e6cb8D13EF1fFEc1a945b) | ERC-8004 identity registry |
| **CREDEndorsementRegistry** | *Pending (requires token)* | Agent-to-agent trust endorsements |
| **CREDStaking** | *Pending (requires token)* | Token staking with slashing |
| **CREDToken** | *Not yet deployed* | ERC-20 governance token |
| **QueryFeeRouter** | *Pending (requires token)* | API query fee distribution |

**EAS (Ethereum Attestation Service):** [`0x4200000000000000000000000000000000000021`](https://basescan.org/address/0x4200000000000000000000000000000000000021)

---

## CREDScoreRegistry

The core contract for storing and querying agent reputation scores. Extends the ERC-8004 Reputation Registry pattern with credit ratings and a trust domain.

### Enums

```solidity
/// Credit rating tiers, mirroring traditional credit agency conventions.
enum CreditRating { D, CCC, B, BB, BBB, A, AA, AAA }
```

### Structs

```solidity
struct DomainScore {
    uint16 score;           // 0-10000 (2-decimal precision: 8742 = 87.42)
    uint16 confidence;      // 0-10000
    uint48 lastUpdated;     // Unix timestamp
    bytes32 easUID;         // EAS attestation UID
}

struct AgentProfile {
    uint256 erc8004Id;        // Linked ERC-8004 identity token
    uint16 overallScore;      // Aggregate score
    uint16 overallConfidence; // Aggregate confidence
    uint256 totalDataPoints;  // Cumulative publications
    uint48 firstScored;       // Registration timestamp
    uint48 lastUpdated;       // Last update timestamp
}
```

### Domain Keys

Domains are identified by `bytes32` keys:

```solidity
bytes32 public constant TRADING    = keccak256("trading");
bytes32 public constant PREDICTION = keccak256("prediction");
bytes32 public constant SOCIAL     = keccak256("social");
bytes32 public constant BUILDER    = keccak256("builder");
bytes32 public constant CONTENT    = keccak256("content");
bytes32 public constant TRUST      = keccak256("trust");
```

### Reading Scores

```solidity
/// @notice Returns the overall score, confidence, and last update timestamp.
function getScore(address agent)
    external view
    returns (uint16 score, uint16 confidence, uint48 lastUpdated);

/// @notice Returns a single domain score.
function getDomainScore(address agent, bytes32 domain)
    external view
    returns (DomainScore memory);

/// @notice Returns the full profile with all domain scores.
function getFullProfile(address agent)
    external view
    returns (
        uint16 overallScore,
        uint16 overallConfidence,
        bytes32[] memory domains,
        DomainScore[] memory domainScores
    );

/// @notice Check if an agent is registered.
function isRegistered(address agent) external view returns (bool);

/// @notice Verify an EAS attestation for a domain score.
function verifyAttestation(address agent, bytes32 domain, bytes32 easUID)
    external view
    returns (bool);
```

### Credit Rating Functions

```solidity
/// @notice Converts a uint16 score (0-10000) to a CreditRating tier.
/// AAA: 9000+, AA: 8000-8999, A: 7000-7999, BBB: 6000-6999,
/// BB: 5000-5999, B: 4000-4999, CCC: 3000-3999, D: <3000
function scoreToRating(uint16 score) public pure returns (CreditRating);

/// @notice Returns the credit rating for an agent based on their overall score.
function getCreditRating(address agent) external view returns (CreditRating);

/// @notice Returns a comprehensive risk assessment for an agent.
function getRiskAssessment(address agent)
    external view
    returns (
        CreditRating rating,
        uint16 overallScore,
        uint16 overallConfidence,
        uint48 lastUpdated,
        uint256 totalDataPoints,
        uint48 firstScored
    );
```

### Registration

```solidity
/// @notice Register as an agent by linking to an ERC-8004 identity.
/// @param erc8004Id The ERC-8004 identity token ID.
function registerAgent(uint256 erc8004Id) external;
```

### Score Publishing (Scorer Only)

```solidity
/// @notice Publish scores for a registered agent across one or more domains.
/// @dev Only callable by an authorized scorer when not paused.
///      Emits CreditRatingChanged when the score crosses a tier boundary.
function publishScore(
    address agent,
    uint16 overallScore,
    uint16 overallConfidence,
    bytes32[] calldata domains,
    uint16[] calldata scores,
    uint16[] calldata confidences,
    bytes32[] calldata easUIDs
) external onlyScorer whenNotPaused;
```

### Events

```solidity
event AgentRegistered(address indexed agent, uint256 erc8004Id);
event ScorePublished(address indexed agent, uint16 overallScore, uint16 confidence);
event CreditRatingChanged(address indexed agent, CreditRating previousRating, CreditRating newRating);
event ScorerAdded(address indexed scorer);
event ScorerRemoved(address indexed scorer);
```

### Admin

```solidity
function addScorer(address scorer) external onlyOwner;
function removeScorer(address scorer) external onlyOwner;
function pause() external onlyOwner;
function unpause() external onlyOwner;
function setIdentityRegistry(address _identityRegistry) external onlyOwner;
```

---

## CREDEndorsementRegistry

Agent-to-agent trust endorsements with optional CREDZ token staking. Endorsements are consumed by the off-chain EigenTrust scorer to compute the "trust" domain score.

### Structs

```solidity
struct Endorsement {
    address endorser;
    address endorsee;
    uint8 weight;           // 1-100
    uint256 stakedAmount;   // optional CREDZ staked behind endorsement
    uint48 createdAt;
    bool isActive;
}
```

### Endorsement Functions

```solidity
/// @notice Endorse another agent. Requires minimum score on the endorser.
function endorse(address endorsee, uint8 weight) external;

/// @notice Endorse with CREDZ tokens staked behind the endorsement.
function endorseWithStake(address endorsee, uint8 weight, uint256 stakeAmount) external;

/// @notice Revoke an endorsement. Returns any staked tokens.
function revokeEndorsement(address endorsee) external;

/// @notice Add additional stake to an existing endorsement.
function addStakeToEndorsement(address endorsee, uint256 amount) external;
```

### View Functions

```solidity
function getEndorsement(address endorser, address endorsee) external view returns (Endorsement memory);
function getEndorsersOf(address agent) external view returns (address[] memory);
function getEndorsingBy(address agent) external view returns (address[] memory);
function getEndorsementCount(address agent) external view returns (uint256 received, uint256 given);
function isEndorsing(address endorser, address endorsee) external view returns (bool);
```

### Events

```solidity
event EndorsementCreated(address indexed endorser, address indexed endorsee, uint8 weight, uint256 stakedAmount);
event EndorsementRevoked(address indexed endorser, address indexed endorsee);
event StakeAdded(address indexed endorser, address indexed endorsee, uint256 amount);
```

### Errors

```solidity
error InsufficientScore();   // Endorser's score is below minimum threshold
error SelfEndorsement();     // Cannot endorse yourself
error AlreadyEndorsed();     // Duplicate endorsement
error NotEndorsed();         // Revoking a non-existent endorsement
error InvalidWeight();       // Weight must be 1-100
error ZeroAddress();         // Cannot endorse the zero address
```

### Configuration

| Property | Value |
|----------|-------|
| **Min endorser score** | 3000 (configurable by owner) |
| **Weight range** | 1-100 |
| **Security** | ReentrancyGuard, Ownable |

---

## CREDZGate

Abstract contract providing score-gating modifiers that any protocol can inherit to gate functionality behind CREDZ reputation requirements. Each modifier performs a single STATICCALL + comparison (~2600 gas overhead).

See [Score Gating](score-gating.md) for a full integration guide.

```solidity
abstract contract CREDZGate {
    CREDScoreRegistry public immutable credScoreRegistry;

    // Custom errors with full context for debugging
    error ScoreBelowMinimum(address agent, uint16 score, uint16 required);
    error DomainScoreBelowMinimum(address agent, bytes32 domain, uint16 score, uint16 required);
    error RatingBelowMinimum(address agent, CreditRating rating, CreditRating required);
    error TrustScoreBelowMinimum(address agent, uint16 score, uint16 required);

    constructor(address _scoreRegistry);

    modifier requireMinScore(address agent, uint16 minScore);
    modifier requireMinDomainScore(address agent, bytes32 domain, uint16 minScore);
    modifier requireMinRating(address agent, CREDScoreRegistry.CreditRating minRating);
    modifier requireMinTrustScore(address agent, uint16 minScore);
}
```

---

## CREDStaking

> **Note:** The staking system is currently under development. The contract is deployed but the dashboard UI is not yet available.

Self-staking with EigenLayer-style slashing. Agents stake CREDZ tokens to back their reputation.

See [Staking](staking.md) for details.

### Constants

```solidity
uint256 public constant COOLDOWN_PERIOD = 30 days;
```

### Functions

```solidity
function stake(uint256 amount) external nonReentrant;
function beginUnstake(uint256 amount) external nonReentrant;
function completeUnstake() external nonReentrant;
function getStake(address agent) external view returns (StakeInfo memory);
function getStakedAmount(address agent) external view returns (uint256);
function slash(address agent, uint256 amount, bytes calldata proof) external nonReentrant onlySlasher;
```

### Events

```solidity
event Staked(address indexed agent, uint256 amount);
event UnstakeInitiated(address indexed agent, uint256 amount, uint48 completionTime);
event Unstaked(address indexed agent, uint256 amount);
event Slashed(address indexed agent, uint256 amount, bytes proof);
```

---

## CREDToken

Standard ERC-20 with burn functionality. Total supply: **1,000,000,000 CREDZ**.

```solidity
contract CREDToken is ERC20, ERC20Burnable, Ownable {
    uint256 public constant TOTAL_SUPPLY = 1_000_000_000 * 10 ** 18;

    constructor() ERC20("CRED", "CRED") Ownable(msg.sender) {
        _mint(msg.sender, TOTAL_SUPPLY);
    }
}
```

Built on OpenZeppelin v5 contracts. Supports standard `transfer`, `approve`, `burn`, and `burnFrom`.

---

## Integration Example

### Solidity — Gate with CREDZGate (Recommended)

```solidity
import {CREDZGate} from "@credz/contracts/CREDZGate.sol";
import {CREDScoreRegistry} from "@credz/contracts/CREDScoreRegistry.sol";

contract MyProtocol is CREDZGate {
    constructor(address registry) CREDZGate(registry) {}

    // Require overall score >= 70.00
    function basicAction()
        external
        requireMinScore(msg.sender, 7000)
    {
        // ...
    }

    // Require BBB rating or higher
    function borrow()
        external
        requireMinRating(msg.sender, CREDScoreRegistry.CreditRating.BBB)
    {
        // ...
    }

    // Require trading domain score >= 60.00 AND trust score >= 50.00
    function leveragedTrade()
        external
        requireMinDomainScore(msg.sender, credScoreRegistry.TRADING(), 6000)
        requireMinTrustScore(msg.sender, 5000)
    {
        // ...
    }
}
```

### Solidity — Direct Registry Reads

```solidity
import {ICREDScoreRegistry} from "@credz/interfaces/ICREDScoreRegistry.sol";

contract ReputationGated {
    ICREDScoreRegistry constant CREDZ =
        ICREDScoreRegistry(0xa8A012d06163630e0D5135d9aE6e61DfBaE06622);

    function isTrusted(address agent) external view returns (bool) {
        (uint16 score,,) = CREDZ.getScore(agent);
        return score >= 7000;
    }

    function getCreditRating(address agent) external view returns (ICREDScoreRegistry.CreditRating) {
        return CREDZ.getCreditRating(agent);
    }
}
```

### viem — Read Score from Frontend

```typescript
import { createPublicClient, http } from "viem";
import { base } from "viem/chains";

const client = createPublicClient({
  chain: base,
  transport: http(),
});

const [score, confidence, lastUpdated] = await client.readContract({
  address: "0xa8A012d06163630e0D5135d9aE6e61DfBaE06622",
  abi: [
    {
      name: "getScore",
      type: "function",
      stateMutability: "view",
      inputs: [{ name: "agent", type: "address" }],
      outputs: [
        { name: "score", type: "uint16" },
        { name: "confidence", type: "uint16" },
        { name: "lastUpdated", type: "uint48" },
      ],
    },
  ],
  functionName: "getScore",
  args: ["0x1234567890abcdef1234567890abcdef12345678"],
});

console.log(`Score: ${score / 100}, Confidence: ${confidence / 100}%`);
```
