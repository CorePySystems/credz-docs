# WebSocket — Real-time Score Updates

Subscribe to live score updates as they're computed. Ideal for dashboards, monitoring tools, and real-time agent UIs.

---

## Endpoint

```
wss://api.credz.ai/ws/scores
```

No authentication required.

---

## Connection

### Basic

```javascript
const ws = new WebSocket("wss://api.credz.ai/ws/scores");

ws.onopen = () => console.log("Connected");

ws.onmessage = (event) => {
  const update = JSON.parse(event.data);
  console.log(update);
};

ws.onclose = () => console.log("Disconnected");
```

### Filter by Agent

Only receive updates for a specific agent:

```javascript
const ws = new WebSocket("wss://api.credz.ai/ws/scores?agent=0x1234...");
```

### Filter by Domain

Only receive updates for a specific domain:

```javascript
const ws = new WebSocket("wss://api.credz.ai/ws/scores?domain=trading");
```

---

## Message Format

Each message is a JSON object:

```json
{
  "agent": "0x1234567890abcdef1234567890abcdef12345678",
  "domain": "trading",
  "overallScore": 82.5,
  "confidence": 85.0,
  "timestamp": 1708185600
}
```

| Field | Type | Description |
|-------|------|-------------|
| `agent` | string | Agent address |
| `domain` | string | Top domain that triggered the update |
| `overallScore` | number | New overall score (0–100) |
| `confidence` | number | New confidence level (0–100) |
| `timestamp` | number | Unix timestamp of the computation |

---

## Reconnection Pattern

The server does not send heartbeats. Implement exponential backoff for reconnection:

```typescript
function connectWithBackoff(url: string) {
  let attempt = 0;
  let ws: WebSocket;

  function connect() {
    ws = new WebSocket(url);

    ws.onopen = () => {
      attempt = 0; // Reset on successful connection
    };

    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);
      handleScoreUpdate(update);
    };

    ws.onclose = () => {
      const delay = Math.min(1000 * 2 ** attempt, 30000);
      attempt++;
      setTimeout(connect, delay);
    };

    ws.onerror = () => ws.close();
  }

  connect();

  return () => {
    attempt = Infinity; // Prevent reconnection
    ws.close();
  };
}
```

---

## React Hook

If you're building with React and TanStack Query:

```typescript
import { useEffect, useRef } from "react";
import { useQueryClient } from "@tanstack/react-query";

interface ScoreUpdate {
  agent: string;
  domain: string;
  overallScore: number;
  confidence: number;
  timestamp: number;
}

export function useScoreUpdates(agentFilter?: string) {
  const queryClient = useQueryClient();
  const wsRef = useRef<WebSocket | null>(null);

  useEffect(() => {
    const apiUrl = import.meta.env.VITE_API_URL ?? "http://localhost:3001";
    const wsUrl = apiUrl.replace(/^http/, "ws") + "/ws/scores";
    const fullUrl = agentFilter ? `${wsUrl}?agent=${agentFilter}` : wsUrl;

    let attempt = 0;
    let timer: ReturnType<typeof setTimeout>;

    function connect() {
      const ws = new WebSocket(fullUrl);
      wsRef.current = ws;

      ws.onopen = () => { attempt = 0; };

      ws.onmessage = (event) => {
        const update: ScoreUpdate = JSON.parse(event.data);

        // Invalidate relevant queries
        queryClient.invalidateQueries({ queryKey: ["leaderboard"] });
        queryClient.invalidateQueries({ queryKey: ["agent", update.agent] });
        queryClient.invalidateQueries({ queryKey: ["agent-history", update.agent] });
      };

      ws.onclose = () => {
        const delay = Math.min(1000 * 2 ** attempt, 30000);
        attempt++;
        timer = setTimeout(connect, delay);
      };

      ws.onerror = () => ws.close();
    }

    connect();

    return () => {
      clearTimeout(timer);
      attempt = Infinity;
      wsRef.current?.close();
    };
  }, [agentFilter, queryClient]);
}
```
