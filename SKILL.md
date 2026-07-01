---
name: daily-enterprise-agents-grounded-research
category: research
tags:
  - ai-agents
  - enterprise
  - market-trends
  - buyer-sentiment
  - grounded-data
description: Collects daily grounded, source-checked industry trend findings and buyer/user sentiment reports for Enterprise Agents, returning raw structured JSON only.
metadata:
  forge:
    requires:
      bins:
        - curl
      env:
        one_of:
          - TAVILY_API_KEY
        required:
          - TAVILY_ENDPOINT
        optional:
          - DEFAULT_LOOKBACK_DAYS
          - LOCALE
    egress_domains:
      - "$TAVILY_ENDPOINT"
    denied_tools:
      - http_request
    timeout_hint: 300
---

# Daily Enterprise Agents Grounded Research

Collects two separate sets of grounded information every day for downstream script generation: (1) AI/industry trend findings and (2) buyer/user sentiment reports. This skill emphasizes traceability: every entry includes a real, checkable URL source and a publish date, with strict product availability governance.

## Tool: daily_grounded_research

Performs web search for (A) enterprise pain points, competitor agent activity, AI agent adoption/ROI stats, and real reported incidents; and (B) buyer/user sentiment on AI agents in production from named sources (public forum threads, review sites, and named survey/report publications). Filters out anonymous/unverifiable opinions and never fabricates stats or quotes.

**Input:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| date | string | No | Target date for the report in `YYYY-MM-DD` format (default: today in UTC) |
| lookback_days | integer | No | How far back to search for newer sources (default: from $DEFAULT_LOOKBACK_DAYS or 30) |
| locales | string | No | Comma-separated locale hints (e.g. `en-US,en-GB`) to bias search results |
| include_competitor_activity | boolean | No | Whether to specifically search competitor agent/product activity (default: true) |
| min_entries_per_section | integer | No | Minimum number of findings to return per section (default: 5) |
| max_entries_per_section | integer | No | Maximum number of findings to return per section (default: 15) |
| filter_out_unnamed_sources | boolean | No | If true, discard entries that do not clearly cite a named publication/report/post/thread (default: true) |
| product_constants_mode | string | No | Must be `enterprise-agents` (validates static fact mapping usage). Default: `enterprise-agents` |

**Output:**

```json
{
  "date": "YYYY-MM-DD",
  "trend_findings": [
    {
      "finding": "1-sentence summary",
      "source": "url",
      "date": "publish date as YYYY-MM-DD (or null if not stated)",
      "relevant_agent": "Rely | Infil | Code | Cost | Workflow",
      "pain_point": "what this implies teams struggle with",
      "matching_product_fact": "static fact addressing this, if any"
    }
  ],
  "consumer_reports": [
    {
      "finding": "1-sentence summary of sentiment or stat",
      "source": "url",
      "date": "publish date as YYYY-MM-DD (or null if not stated)",
      "sentiment": "positive | negative | mixed | concern",
      "relevant_agent": "Rely | Infil | Code | Cost | Workflow | general",
      "matching_product_fact": "static fact addressing this, if any"
    }
  ]
}
```

**Examples:**

| User Request | Tool Input |
|---|---|
| “Generate today’s grounded trend and consumer sentiment JSON for Enterprise Agents.” | `{ "date": "2026-07-01" }` |
| “Search last 14 days and return at least 5 entries per section.” | `{ "lookback_days": 14, "min_entries_per_section": 5 }` |
| “Bias to US English sources; include competitor activity.” | `{ "locales": "en-US", "include_competitor_activity": true }` |
| “Return up to 10 items per section; discard sources without clear names.” | `{ "max_entries_per_section": 10, "filter_out_unnamed_sources": true }` |
| “I only want the workflow/governance-related mapping where applicable.” | `{ "min_entries_per_section": 5 }` |
| “Use default lookback and return JSON only.” | `{}` |
| “Target a specific date and broaden the search to 45 days.” | `{ "date": "2026-06-15", "lookback_days": 45 }` |
| “Prefer sources that include real figures (surveys/reports).” | `{ "filter_out_unnamed_sources": true }` |

**Detection heuristics** — use this tool when the agent/user requests:
- Daily/recurring collection of “grounded” market trends plus buyer sentiment
- Enterprise pain points, agent adoption/ROI, incidents (outages, breaches, cost overruns)
- Buyer sentiment on AI agents in production, trust concerns, compliance, hallucination, job displacement fears
- Named publications, surveys (e.g., Gartner/McKinsey-style), or review/feedback sites (G2/Capterra-style)
- A requirement for strictly source-checked raw structured JSON output

## Safety Constraints

- **NO fabrication / hallucination**: Never invent customer names, quotes, review scores, or survey numbers. If a stat/quote is not verifiable from the source, discard it.
- **Product availability governance**:
  - Never mark **Code** or **Cost** as available.
  - Only map `relevant_agent` to `Rely`/`Infil` when there is evidence tied to those capabilities being in-market; otherwise use `general` or `Workflow` as appropriate.
- **Source traceability**:
  - Every entry in both lists must include a real, checkable `source` URL.
  - Discard anonymous/unverifiable opinions without a traceable named post/thread/publication.
- **Distinguish sections**:
  - `trend_findings` must be market/industry/incident/adoption oriented.
  - `consumer_reports` must be buyer/user sentiment or named survey/report feedback.
- **No merging**: Do not combine both types into one list or blur categories.
- **No mutating behavior**: This skill only returns structured data; it must not create/update anything.

## Important Notes

- `date` fields: if the source publication date is not explicit, return `null` rather than guessing.
- `matching_product_fact`: choose from the provided static facts only (e.g., “structured permissions”, “audit trail”, “human approval workflows”, “deep infra integration”, “production environments”); if none clearly match, return `""` (empty string) or omit the field value as `null` (downstream tolerant).
- `relevant_agent` values allowed by schema: `Rely | Infil | Code | Cost | Workflow | general`.
- Search filters for buyer sentiment should prioritize named accounts/public threads and named reports; do not keep uncredentialed “anonymous says” content.