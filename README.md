# Enterprise Agents — Info Feed Bot

Daily research agent that gathers AI trend data and consumer sentiment relevant to the **Enterprise Agents** product line, and outputs structured JSON for a downstream **video ad script generator**.

This bot does **not** write scripts, hooks, or ad copy. It only produces grounded, sourced research data that another agent consumes.

---

## What it does

Every 24 hours (or on-demand), the bot:

1. Runs web searches via the **Tavily API**
2. Collects two categories of findings:
   - **Trend findings** — industry/market signals (competitor activity, adoption stats, incidents, pain points)
   - **Consumer reports** — buyer/user sentiment (reviews, forum feedback, survey data, trust concerns)
3. Maps each finding to a product/agent and a relevant static product fact
4. Outputs everything as dated, structured JSON

---

## Product facts (static — do not regenerate)

The bot has hardcoded, ground-truth knowledge of the product it's researching for. This never changes with the daily research cycle.

| Agent | Category | Status |
|---|---|---|
| Rely | Reliability | Available |
| Infil | Security / pentesting | Available |
| Code | Code quality | Coming soon |
| Cost | Cost optimization | Coming soon |

**Differentiators:** structured permissions, audit trails, human approval workflows, deep infra integration (Kubernetes, GitHub, Datadog, Prometheus, cloud platforms, databases, internal APIs).

⚠️ The bot must never claim "Coming soon" products are available, and must never fabricate customer names, quotes, or statistics.

---

## Trigger

- Scheduled: every 24 hours
- Manual: on-demand, before a script generation run

---

## Search targets

**Trend findings:**
- Enterprise pain points (alert fatigue, slow incident response, security backlog, cloud overspend)
- Competitor AI agent launches/activity
- Adoption/ROI/trust stats for enterprise AI agents
- Real, publicly reported incidents (outages, breaches, cost overruns)

**Consumer reports:**
- Buyer/user sentiment (Reddit, forums, review sites, G2/Capterra-style feedback)
- Trust/adoption hesitancy (safety, hallucination, compliance, job displacement fears)
- Praise or complaints about existing agent tools in the same categories
- Named survey/analyst reports (e.g. Gartner, McKinsey) with real figures

**Filter rule:** discard anything unsourced or unverifiable. Every entry must trace back to a real URL and date.

---

## Output format

```json
{
  "date": "YYYY-MM-DD",
  "trend_findings": [
    {
      "finding": "1-sentence summary",
      "source": "url",
      "date": "publish date",
      "relevant_agent": "Rely | Infil | Code | Cost | Workflow",
      "pain_point": "what this implies teams struggle with",
      "matching_product_fact": "static fact addressing this, if any"
    }
  ],
  "consumer_reports": [
    {
      "finding": "1-sentence summary of sentiment or stat",
      "source": "url",
      "date": "publish date",
      "sentiment": "positive | negative | mixed | concern",
      "relevant_agent": "Rely | Infil | Code | Cost | Workflow | general",
      "matching_product_fact": "static fact addressing this, if any"
    }
  ]
}
```

`trend_findings` and `consumer_reports` are kept as **separate arrays** — never merged — so the script generator can pull from either independently.

---

## Guardrails

- Every finding requires a real, checkable source (URL + date)
- Never mark Code or Cost as "available"
- Never invent customer names, quotes, review scores, or survey numbers
- Trend and consumer data stay in separate lists, not combined
- If no relevant findings exist for a category on a given day, output an empty array — don't omit the key or fabricate a filler entry

---

## Downstream usage

This bot's JSON output is the sole input to the **script generator agent**. It does not:
- Write hooks, taglines, or CTAs
- Decide video format or length
- Make creative or structural decisions about the ad

Those responsibilities belong entirely to the script generator, which consumes this bot's output.

---

## Dependencies

- Tavily API (web search)
- Access to static product fact sheet (embedded in bot config, updated manually when product info changes — not part of the daily research cycle)
