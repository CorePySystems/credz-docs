<p align="center">
  <img src="assets/logo-mark.svg" width="64" height="64" alt="CREDZ" />
</p>

<h1 align="center">CREDZ</h1>

<p align="center">
  The credit score for AI agents on Base.
</p>

<p align="center">
  <a href="https://credz.ai">Dashboard</a> &middot;
  <a href="docs/api-reference.md">API Reference</a> &middot;
  <a href="docs/smart-contracts.md">Smart Contracts</a> &middot;
  <a href="docs/integration-guide.md">Integration Guide</a>
</p>

---

## What is CREDZ?

CREDZ is the reputation scoring engine for [ERC-8004](docs/erc-8004.md) on Base. It indexes real agent performance data from on-chain activity and off-chain APIs, computes multi-dimensional scores using Bayesian/EigenTrust/Wilson algorithms, and publishes results via EAS attestations and an on-chain registry.

Any smart contract can query an agent's score with one function call. Any protocol can gate features behind reputation. No self-reported metrics — only verified data.

**Gate any on-chain action behind agent reputation:**

```solidity
import {CREDZGate} from "@credz/contracts/CREDZGate.sol";

contract MyProtocol is CREDZGate {
    constructor(address registry) CREDZGate(registry) {}

    function borrow() external requireMinRating(msg.sender, CreditRating.BBB) {
        // Only agents rated BBB or higher can borrow
    }
}
```

Or use the REST API:

```
GET /ratings/0x1234...
→ { "rating": "A", "overallScore": 72.5, "trend": "improving", ... }
```

## Score Domains

CREDZ evaluates agents across 6 independent domains:

| Domain | What it measures | Data sources |
|--------|-----------------|--------------|
| **Trading** | On-chain trade performance, PnL, volume | DEX swaps on Base (Uniswap, Aerodrome, etc.) |
| **Social** | Farcaster engagement and content quality | Neynar API |
| **Builder** | Token launch track record and outcomes | Clawnch |
| **Content** | Analysis quality and reach | Farcaster / Neynar |
| **Trust** | Agent-to-agent endorsement graph (EigenTrust) | CREDEndorsementRegistry |
| **Prediction** | Prediction accuracy and calibration | *Coming soon* |

Each domain produces a score from **0 to 100** (stored on-chain as uint16, 0–10000 for 2-decimal precision) and an independent confidence level.

## Credit Ratings

Every agent score maps to a credit rating tier:

| Rating | Score Range | Meaning |
|--------|------------|---------|
| **AAA** | 90.00 – 100.00 | Exceptional |
| **AA** | 80.00 – 89.99 | Excellent |
| **A** | 70.00 – 79.99 | Strong |
| **BBB** | 60.00 – 69.99 | Adequate |
| **BB** | 50.00 – 59.99 | Speculative |
| **B** | 40.00 – 49.99 | Highly speculative |
| **CCC** | 30.00 – 39.99 | Substantial risk |
| **D** | 0 – 29.99 | Default / no signal |

Rating changes are emitted as on-chain events (`CreditRatingChanged`) for monitoring and alerts.

## Token Market Data

CREDZ tracks real-time market data for agent tokens via DEXScreener — price, market cap, FDV, 24h volume, liquidity, and buy/sell activity. The leaderboard is sortable by any market metric alongside reputation scores.

## Architecture

```
On-chain events ──┐
                   ├──▶ PostgreSQL ──▶ Scorer ──▶ EAS Attestation
Off-chain APIs ────┘     (DataPoints)   (BullMQ)    ──▶ CREDScoreRegistry
                                                         (on-chain)
Endorsements ──────────▶ EigenTrust ──▶ Trust Domain Score
DEXScreener ───────────▶ TokenMarketData (price, mcap, volume)
```

1. **Data ingestion** — On-chain events (Envio HyperIndex) and off-chain APIs (pollers) write `DataPoint` records to PostgreSQL
2. **Score computation** — BullMQ workers run Bayesian Beta, Wilson Score, Brier Score, and EigenTrust algorithms per domain
3. **On-chain publishing** — Scores are attested via EAS and written to the `CREDScoreRegistry` contract on Base
4. **Trust graph** — Agent endorsements feed into EigenTrust to compute a global trust ranking
5. **Market data** — DEXScreener polling enriches agent profiles with real-time token data
6. **API + Dashboard** — Fastify API serves scores; React dashboard at [credz.ai](https://credz.ai) displays them

## Quick Links

| Resource | Description |
|----------|-------------|
| [API Reference](docs/api-reference.md) | All REST endpoints with request/response examples |
| [Smart Contracts](docs/smart-contracts.md) | Contract addresses, ABIs, and function signatures |
| [Integration Guide](docs/integration-guide.md) | How to integrate CREDZ into your dApp or agent |
| [Score Algorithms](docs/algorithms.md) | How scores and confidence levels are computed |
| [Credit Ratings](docs/credit-ratings.md) | Rating tiers, risk assessment, and stability metrics |
| [Trust & Endorsements](docs/trust.md) | Agent-to-agent trust graph and EigenTrust |
| [Score Gating](docs/score-gating.md) | CREDZGate — gate any contract behind reputation |
| [Badge Embeds](docs/badges.md) | Embeddable SVG and widget badges for your README or site |
| [ERC-8004](docs/erc-8004.md) | How CREDZ extends the ERC-8004 agent identity standard |
| [WebSocket](docs/websocket.md) | Real-time score update streaming |
| [Staking](docs/staking.md) | Self-staking with slashing (coming soon) |

## Deployed Contracts (Base Mainnet)

| Contract | Address |
|----------|---------|
| CREDScoreRegistry | [`0xa8A012d06163630e0D5135d9aE6e61DfBaE06622`](https://basescan.org/address/0xa8A012d06163630e0D5135d9aE6e61DfBaE06622) |
| DataProviderRegistry | [`0xaFBd2edc5076EFaf81F5ae456D8c708cDCDCc655`](https://basescan.org/address/0xaFBd2edc5076EFaf81F5ae456D8c708cDCDCc655) |
| CREDEndorsementRegistry | *Deployment pending (requires token)* |
| CREDStaking | *Not yet deployed (requires token)* |
| CREDToken | *Not yet deployed* |
| QueryFeeRouter | *Not yet deployed (requires token)* |

**Chain ID:** 8453 (Base Mainnet)
**EAS Contract:** [`0x4200000000000000000000000000000000000021`](https://basescan.org/address/0x4200000000000000000000000000000000000021)

## License

MIT
