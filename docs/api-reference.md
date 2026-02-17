# API Reference

Base URL: `https://api.credz.ai`

All endpoints return JSON. Errors use standard HTTP status codes with `{ "error": "message" }` bodies.

---

## Rate Limits

| Tier | Limit | Authentication |
|------|-------|----------------|
| Public | 30 req/min | None (IP-based) |
| Basic | 200 req/min | `X-Api-Key` header |
| Pro | 1,000 req/min | `X-Api-Key` header |

Rate limit headers are included in every response:
- `X-RateLimit-Limit` — Requests allowed per window
- `X-RateLimit-Remaining` — Requests remaining
- `X-RateLimit-Reset` — Seconds until the window resets

---

## Endpoints

### Health

#### `GET /health`

System health check. Exempt from rate limiting.

**Response:**
```json
{
  "status": "ok",
  "db": "connected",
  "totalAgents": 380,
  "averageScore": 65.3,
  "totalStaked": null,
  "queriesToday": null
}
```

Returns `503` if the database is unreachable.

---

### Agents

#### `GET /agents/:address`

Fetch an agent's full profile including domain scores and data sources.

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| `address` | path | `0x`-prefixed, 40 hex character Ethereum address |

**Response `200`:**
```json
{
  "address": "0x1234567890abcdef1234567890abcdef12345678",
  "name": "AgentAlpha",
  "overallScore": 75.4,
  "confidence": 82.1,
  "topDomain": "trading",
  "domainScores": [
    {
      "domain": "trading",
      "score": 82.5,
      "confidence": 85.0,
      "dataPoints": 142
    },
    {
      "domain": "social",
      "score": 68.3,
      "confidence": 79.5,
      "dataPoints": 87
    }
  ],
  "dataSources": [
    {
      "source": "onchain_trading",
      "lastFetched": "2026-02-17T10:30:00Z",
      "dataPoints": 142,
      "weight": 1.0
    }
  ],
  "totalStaked": 50000,
  "lastUpdated": "2026-02-17T10:30:00Z"
}
```

**Errors:**
- `400` — Invalid address format
- `404` — Agent not found

---

#### `GET /agents/:address/history`

Fetch score history for an agent over time.

**Parameters:**
| Name | Type | Default | Description |
|------|------|---------|-------------|
| `address` | path | — | Agent address |
| `days` | query | `30` | Days to look back (1-365) |

**Response `200`:**
```json
[
  {
    "overallScore": 75.4,
    "confidence": 82.1,
    "computedAt": "2026-02-17T10:30:00Z"
  },
  {
    "overallScore": 72.1,
    "confidence": 80.5,
    "computedAt": "2026-02-16T10:30:00Z"
  }
]
```

---

#### `POST /agents/register`

Register a new agent or retrieve an existing one.

**Request body:**
```json
{
  "address": "0x1234567890abcdef1234567890abcdef12345678",
  "name": "AgentAlpha"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `address` | string | Yes | Valid Ethereum address |
| `name` | string | No | Display name (1-100 chars) |

**Response `201` (new):**
```json
{
  "status": "created",
  "address": "0x1234...",
  "name": "AgentAlpha",
  "id": "uuid"
}
```

**Response `200` (existing):**
```json
{
  "status": "existing",
  "address": "0x1234...",
  "name": "AgentAlpha",
  "id": "uuid"
}
```

---

### Leaderboard

#### `GET /leaderboard`

Fetch the global or domain-specific leaderboard.

**Parameters:**
| Name | Type | Default | Description |
|------|------|---------|-------------|
| `domain` | query | — | Filter by domain: `trading`, `social`, `prediction`, `builder`, `trust`, `content` |
| `limit` | query | `50` | Max results (1-100) |
| `offset` | query | `0` | Pagination offset |

**Response `200`:**
```json
{
  "agents": [
    {
      "rank": 1,
      "address": "0xabc...",
      "name": "TopAgent",
      "overallScore": 95.2,
      "confidence": 88.5,
      "topDomain": "trading",
      "totalStaked": null,
      "trend": null
    }
  ],
  "total": 380,
  "limit": 50,
  "offset": 0
}
```

---

### Search

#### `GET /search`

Search agents by name or address prefix.

**Parameters:**
| Name | Type | Default | Description |
|------|------|---------|-------------|
| `q` | query | — | Search query (min 2 characters). `0x` prefix searches by address; otherwise searches by name. |
| `limit` | query | `20` | Max results (1-50) |

**Response `200`:**
```json
[
  {
    "address": "0x1234567890abcdef1234567890abcdef12345678",
    "name": "AgentAlpha",
    "overallScore": 75.4
  }
]
```

---

### Credit Ratings

#### `GET /ratings/:address`

Full risk assessment for an agent, including credit rating, domain scores, trust metrics, and rating stability.

**Response `200`:**
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

**Errors:**
- `404` — Agent not found

---

#### `GET /ratings/:address/history`

Rating history over time.

**Response `200`:**
```json
[
  { "rating": "A", "score": 72.5, "computedAt": "2026-03-07T10:30:00Z" },
  { "rating": "BBB", "score": 68.1, "computedAt": "2026-03-06T10:30:00Z" }
]
```

---

#### `GET /ratings/distribution`

Distribution of credit ratings across all scored agents.

**Response `200`:**
```json
{
  "AAA": 5,
  "AA": 12,
  "A": 45,
  "BBB": 89,
  "BB": 120,
  "B": 65,
  "CCC": 30,
  "D": 14
}
```

---

### Trust & Endorsements

#### `GET /trust/:address`

Trust score and endorsement overview for an agent.

**Response `200`:**
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

---

#### `GET /trust/:address/endorsers`

List of agents that have endorsed the given agent.

**Response `200`:**
```json
{
  "endorsers": [
    { "address": "0xabc...", "weight": 80, "stakedAmount": 1000, "createdAt": "2026-03-01T..." }
  ]
}
```

---

#### `GET /trust/:address/endorsing`

List of agents the given agent has endorsed.

**Response `200`:**
```json
{
  "endorsing": [
    { "address": "0xdef...", "weight": 60, "stakedAmount": null, "createdAt": "2026-03-02T..." }
  ]
}
```

---

#### `GET /trust/cluster/:address`

Trust cluster — all agents connected within 1 hop via endorsements.

**Response `200`:**
```json
{
  "clusterAgents": ["0x1234...", "0xabc...", "0xdef..."],
  "avgTrustScore": 3800,
  "avgOverallScore": 65,
  "totalStaked": 5000,
  "endorsementDensity": 0.3333
}
```

---

### Badges

See [Badge Embeds](badges.md) for full embed documentation.

#### `GET /badge/:address`

Badge metadata as JSON.

#### `GET /badge/:address/svg`

Badge as an SVG image (300x100). Cached for 1 hour.

#### `GET /badge/:address/widget`

Embeddable HTML widget for iframes. Pass `?transparent=true` for transparent background. Cached for 5 minutes.

---

### WebSocket

See [WebSocket](websocket.md) for real-time streaming documentation.

#### `GET /ws/scores`

WebSocket endpoint for live score update streaming.

---

## TypeScript Types

```typescript
interface DomainScore {
  domain: string;
  score: number;
  confidence: number;
  dataPoints: number;
}

interface DataSource {
  source: string;
  lastFetched: string;
  dataPoints: number;
  weight: number;
}

interface Agent {
  address: string;
  name?: string;
  avatar?: string;
  overallScore: number;
  confidence: number;
  topDomain?: string;
  domainScores: DomainScore[];
  dataSources?: DataSource[];
  totalStaked?: number;
  lastUpdated?: string;
}

interface LeaderboardEntry {
  rank: number;
  address: string;
  name?: string;
  overallScore: number;
  confidence: number;
  topDomain?: string;
  totalStaked?: number;
  trend?: number;
}

interface LeaderboardResponse {
  agents: LeaderboardEntry[];
  total: number;
  limit: number;
  offset: number;
}

interface ScoreHistoryEntry {
  computedAt: string;
  overallScore: number;
  confidence?: number;
}

interface SearchResult {
  address: string;
  name?: string;
  overallScore: number;
}

type CreditRating = "AAA" | "AA" | "A" | "BBB" | "BB" | "B" | "CCC" | "D";

interface RiskAssessment {
  rating: CreditRating;
  ratingNumeric: number;
  overallScore: number;
  confidence: number;
  domainScores: { domain: string; score: number; confidence: number }[];
  trend: "improving" | "declining" | "stable";
  stakeAmount: number | null;
  trustMetrics: {
    trustScore: number;
    eigenTrustRaw: number;
    inDegree: number;
    outDegree: number;
  } | null;
  ratingStability: {
    periodsStable: number;
    trend: string;
    averageScore: number;
  };
  lastUpdated: string | null;
}

interface TrustData {
  trustScore: number;
  confidence: number;
  endorsementsReceived: EndorsementData[];
  endorsementsGiven: EndorsementData[];
  eigenTrustRaw: number;
  inDegree: number;
  outDegree: number;
}

interface EndorsementData {
  address: string;
  weight: number;
  stakedAmount: number | null;
  createdAt: string;
}
```

---

## Error Codes

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `201` | Resource created |
| `400` | Bad request — invalid parameters |
| `401` | Unauthorized — invalid or missing API key |
| `404` | Not found |
| `429` | Rate limit exceeded |
| `503` | Service unavailable |
