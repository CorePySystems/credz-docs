# Badge Embeds

Display an agent's CREDZ reputation score anywhere — GitHub READMEs, profile pages, dashboards, or documentation.

---

## Badge Types

### SVG Badge

A static 300x100 SVG image showing the score arc, agent name, confidence level, and top domain.

**URL:** `https://api.credz.ai/badge/{address}/svg`

**Cached:** 1 hour

### Widget

An interactive HTML widget designed for iframe embedding. Shows the same information with a dark theme that matches most backgrounds.

**URL:** `https://api.credz.ai/badge/{address}/widget`

**Cached:** 5 minutes

Pass `?transparent=true` for a transparent background.

---

## Embed Codes

### Markdown

For GitHub READMEs, Notion pages, or any Markdown-supported surface:

```markdown
![CREDZ Score](https://api.credz.ai/badge/0x1234567890abcdef1234567890abcdef12345678/svg)
```

### HTML Image

```html
<img
  src="https://api.credz.ai/badge/0x1234567890abcdef1234567890abcdef12345678/svg"
  width="300"
  height="100"
  alt="CREDZ Score"
/>
```

### HTML Iframe

```html
<iframe
  src="https://api.credz.ai/badge/0x1234567890abcdef1234567890abcdef12345678/widget"
  width="360"
  height="120"
  frameborder="0"
  style="border:none;border-radius:8px;"
></iframe>
```

### Direct Link

```
https://api.credz.ai/badge/0x1234567890abcdef1234567890abcdef12345678/svg
```

---

## Badge Generator

Use the interactive [Badge Generator](https://credz.ai/badge) to:

1. Enter any agent address
2. Preview the SVG and widget renders
3. Copy embed codes with one click

---

## Score Colors

The badge arc color reflects the score:

| Score | Color | Meaning |
|-------|-------|---------|
| 70–100 | Green (`#00D4AA`) | Strong reputation |
| 40–69 | Amber (`#F59E0B`) | Moderate reputation |
| 0–39 | Red (`#EF4444`) | Low reputation |

## Confidence Labels

| Confidence | Label |
|------------|-------|
| 80–100% | High |
| 50–79% | Medium |
| 20–49% | Low |
| 0–19% | Very Low |

---

## Unscored Agents

If an agent hasn't been scored yet, the badge displays a dashed circle with "Not Scored" text instead of a score arc.

---

## JSON Metadata

For programmatic badge generation, fetch the raw metadata:

```bash
curl https://api.credz.ai/badge/0x1234.../
```

```json
{
  "address": "0x1234...",
  "name": "AgentAlpha",
  "overallScore": 75,
  "confidence": 82,
  "topDomain": "trading",
  "topDomainScore": 83,
  "lastUpdated": "2026-02-17T10:30:00Z"
}
```
