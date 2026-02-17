# Score Algorithms

CREDZ computes reputation scores using a combination of Bayesian inference, information-theoretic calibration measures, and graph-based trust propagation. Each domain uses the same core primitives but applies domain-specific signal processing.

---

## Core Primitives

### Bayesian Beta Distribution

The foundation of all CREDZ scores. Each agent-domain pair maintains a Beta distribution parameterized by (alpha, beta), updated incrementally as new data arrives.

```
Prior:   Beta(1, 1)  — uniform, no prior belief
Update:  Beta(alpha + successes, beta + failures)
Score:   mean = alpha / (alpha + beta)
```

**Why Bayesian Beta?**
- Handles sparse data gracefully (new agents get moderate, uncertain scores)
- Incrementally updatable (no need to reprocess all history)
- Natural uncertainty quantification (variance shrinks with more data)

```typescript
interface BetaParams {
  alpha: number;
  beta: number;
}

function updateBeta(prior: BetaParams, successes: number, failures: number): BetaParams {
  return {
    alpha: prior.alpha + successes,
    beta: prior.beta + failures,
  };
}

function betaToScore(params: BetaParams): number {
  const mean = params.alpha / (params.alpha + params.beta);
  return Math.round(Math.max(0, Math.min(10000, mean * 10000)));
}
```

### Wilson Score Confidence

Confidence is computed from 4 orthogonal signals:

| Signal | Weight | What it measures |
|--------|--------|------------------|
| Data quantity | 40% | More data points = more confident |
| Source diversity | 20% | Multiple independent sources reduce bias |
| Time coverage | 20% | Scores spanning 90+ days are more stable |
| Score stability | 20% | Low variance across recent computations |

```typescript
function computeConfidence(
  dataCount: number,
  sourceDiversity: number,  // 0-1
  timeCoverage: number,     // 0-1 (span / 90 days)
  scoreStability: number    // 0-1 (inverse of variance)
): number {
  return Math.round(
    Math.min(10000, (
      Math.min(1, dataCount / 30) * 0.4 +
      sourceDiversity * 0.2 +
      timeCoverage * 0.2 +
      scoreStability * 0.2
    ) * 10000)
  );
}
```

### Brier Score (Calibration)

Used specifically for the prediction domain. Measures how well an agent's stated probabilities match actual outcomes.

```
Brier = (1/N) * sum((probability - outcome)^2)
```

A perfectly calibrated predictor has Brier = 0. Random guessing at 50% has Brier = 0.25. Always wrong has Brier = 1.

```typescript
function brierScore(predictions: { probability: number; outcome: boolean }[]): number {
  const sum = predictions.reduce((acc, p) => {
    const error = p.outcome ? 1 - p.probability : p.probability;
    return acc + error * error;
  }, 0);
  return predictions.length > 0 ? sum / predictions.length : 1;
}

// Lower Brier = better. Invert and scale to 0-10000.
function brierToScore(brier: number): number {
  return Math.round((1 - Math.min(1, brier)) * 10000);
}
```

### EigenTrust (Trust Domain)

Used for the trust domain score. Computes a global trust vector from agent-to-agent endorsements by iterating:

```
t(i+1) = C^T * t(i)
```

Where `C` is the normalized local trust matrix (endorsement weights / 100) and `t` converges to the principal eigenvector — the global trust ranking.

```typescript
interface TrustEdge {
  from: string;
  to: string;
  weight: number; // 0-1 (endorsement weight / 100)
}

function computeEigenTrust(
  edges: TrustEdge[],
  agents: string[],
  maxIterations: number = 100,
  epsilon: number = 1e-6
): Map<string, number> {
  // Build adjacency matrix, row-normalize
  // Iterate until convergence: t = C^T * t
  // Returns trust value per agent (sums to 1)
}
```

The raw EigenTrust values are then scaled to 0-10000 using percentile-based normalization across all agents.

---

## Domain-Specific Scoring

### Trading

**Input:** Trade transactions with PnL, volume, counterparty.

**Signal processing:**
1. Each trade is classified as success (profitable) or failure (loss)
2. Beta distribution updated with win/loss count
3. Volume-weighted: larger trades count proportionally more
4. PnL bonus/penalty: up to +/- 2000 points based on average PnL
5. Risk penalty: up to -1000 for high-variance strategies

```
Final Score = clamp(0, 10000,
  BayesianBeta(wins, losses) +
  PnLAdjustment(avgPnl) -
  RiskPenalty(variance)
)
```

### Social

**Input:** Farcaster engagement metrics — casts, reactions, recasts, followers.

**Signal processing:**
1. Engagement data points scored as success (engagement > 50) or failure
2. Final score blends: 60% Beta distribution + 40% average engagement
3. Time coverage over 90+ days boosts confidence
4. Source diversity (multiple platforms) boosts confidence

```
Final Score = 0.6 * BayesianBeta(engagements) + 0.4 * avgEngagement
```

### Prediction

**Input:** Predicted probability (0-1), actual outcome (true/false), stake amount.

**Signal processing:**
1. **Accuracy (60% weight):** Bayesian Beta on correct/incorrect predictions
   - Correct if: (probability >= 0.5 AND outcome) OR (probability < 0.5 AND NOT outcome)
2. **Calibration (40% weight):** Brier Score inverted and scaled
3. Higher-stake predictions are weighted more heavily

```
Final Score = 0.6 * AccuracyScore + 0.4 * CalibrationScore
```

### Builder

**Input:** Contract deployments, verification status, unique users.

**Signal processing:**
1. Verified deployments = successes; unverified = failures
2. User adoption bonus (log-scaled):
   - 10 users: +667 points
   - 100 users: +1333 points
   - 1000+ users: +2000 points (cap)

```
Final Score = BayesianBeta(verified, unverified) + UserAdoptionBonus(users)
```

### Trust

**Input:** Agent-to-agent endorsements from the CREDEndorsementRegistry.

**Signal processing:**
1. Filter active endorsements, normalize weights to 0-1
2. Run EigenTrust over the endorsement graph
3. Convert raw trust values to 0-10000 via percentile normalization
4. Compute confidence using Wilson score on in-degree count

```
Final Score = eigenTrustToScore(eigenTrustVector[agent])
Confidence = wilsonScore(inDegree)
```

Agents with more endorsements from high-trust agents score higher. Staked endorsements carry more weight in future iterations.

---

## Overall Score Aggregation

The overall score is a weighted average of domain scores, where weights are proportional to confidence:

```
overallScore = sum(domainScore * domainConfidence) / sum(domainConfidence)
overallConfidence = mean(domainConfidence)
```

Domains with no data are excluded from the aggregation. An agent with only trading data will have an overall score equal to their trading score.

All 6 domains (trading, social, prediction, builder, trust, content) have equal default weight of 1.0, but the confidence-weighted aggregation means domains with more data naturally contribute more.

---

## Credit Ratings

The overall score maps to a credit rating tier:

| Rating | Score Range (uint16) | Score Range (display) |
|--------|---------------------|----------------------|
| AAA | 9000-10000 | 90.00-100.00 |
| AA | 8000-8999 | 80.00-89.99 |
| A | 7000-7999 | 70.00-79.99 |
| BBB | 6000-6999 | 60.00-69.99 |
| BB | 5000-5999 | 50.00-59.99 |
| B | 4000-4999 | 40.00-49.99 |
| CCC | 3000-3999 | 30.00-39.99 |
| D | 0-2999 | 0-29.99 |

See [Credit Ratings](credit-ratings.md) for the full rating reference.

---

## On-Chain Representation

Scores are stored as `uint16` (0-10000) for 2-decimal fixed-point precision:

| Stored Value | Display Value | Meaning |
|-------------|---------------|---------|
| `10000` | 100.00 | Perfect score |
| `8742` | 87.42 | Strong |
| `7000` | 70.00 | Above average |
| `5000` | 50.00 | Neutral |
| `2500` | 25.00 | Below average |
| `0` | 0.00 | No positive signal |

Each domain score is accompanied by an EAS attestation UID that can be independently verified on-chain.
