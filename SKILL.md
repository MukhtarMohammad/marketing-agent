---
name: ai-marketing-brain
category: ops
tags:
  - ai
  - marketing
  - video
  - orchestration
  - daily
description: A daily, autonomous brain coordinating seven AI agents to plan, produce, format, publish, and analyze daily marketing videos for YouTube and Instagram. It discovers agents via Agent Cards (A2A) and coordinates communication via a hybrid message bus and REST, maintaining a centralized plan and plan history on a local machine. All agents publish their agent cards; no centralized secret store is required.
metadata:
  forge:
    requires:
      bins:
        - bash
        - curl
        - jq
    env:
      # Credentials for A2A agent coordination provided via environment variables
      required:
        - MARKETING_AGENT_URL      # URL for the Market Research Agent
        - MARKETING_AGENT_TOKEN    # Token for Marketing Agent
        - SCRIPT_GENERATOR_AGENT_URL # URL for the Script Generator Agent
        - SCRIPT_GENERATOR_AGENT_TOKEN # Token for Script Generator Agent
        - CONTENT_GENERATOR_AGENT_URL # URL for the Content Generator Agent
        - CONTENT_GENERATOR_AGENT_TOKEN # Token for Content Generator Agent
      one_of: []
      optional:
        - BRAINS_LOG_LEVEL
    egress_domains:
      - agent-registry.local
      - agent-registry.example.com
      - youtube.googleapis.com
      - graph.instagram.com
      - graph.facebook.com
      - www.googleapis.com
    denied_tools:
      - http_request
    timeout_hint: 3600
---

AI Marketing Brain
A centralized, autonomous orchestrator coordinating seven AI agents to plan, produce, format, publish, and analyze daily marketing videos for YouTube and Instagram. It discovers agents via Agent Cards (A2A) and coordinates communication via a hybrid message bus and REST, maintaining a centralized plan and plan history on a local machine. All agents publish their agent cards; no centralized secret store is required.

Tool: brain_orchestrator
Coordinate, plan, and execute the end-to-end daily marketing workflow by discovering agents via their agent cards, delegating tasks, and aggregating results. Communicate with agents via a message bus and REST endpoints, enforce the defined workflow, and handle retries and alerts on failure.

Input:

| Parameter | Type | Required | Description |

|----------------|--------|----------|-----------------------------------------------------------------------------------------------|

| target_date | string | No | The publish date for the video in YYYY-MM-DD format (default: today). |

| platforms | string | No | Comma-separated list of platforms to publish to (e.g., "youtube,instagram"). Default includes both. |

| topic | string | No | High-level marketing topic for the video (e.g., "Product Launch", "Onboarding"). |

| style | string | No | Creative/style hint (e.g., "brand-safe", "educational", "short-form"). |

| plan_id | string | No | Identifier for this orchestration run (for traceability). |

Output:

JSON object describing the run, including plan steps, agent statuses, results, and audit trail. Example structure:

{
  "execution_id": "exec-2026-07-07-0001",
  "start_time": "2026-07-07T09:00:00Z",
  "target_date": "2026-07-07",
  "platforms": ["youtube", "instagram"],
  "topic": "Product Launch: Platform Alpha",
  "style": "brand-safe",
  "plan_id": "plan-2026-07-07-0001",
  "status": "in_progress",
  "agents": [
    {"name": "market_researcher", "status": "completed", "notes": "Topics identified: features, benefits, pain points"},
    {"name": "script_generator", "status": "in_progress"},
    {"name": "content_generator", "status": "pending"},
    {"name": "consistency_gate", "status": "pending"},
    {"name": "platform_formatter", "status": "pending"},
    {"name": "publisher", "status": "pending"},
    {"name": "analytics_info_scalper", "status": "pending"}
  ],
  "plan": [
    {"step": "discover_agents", "status": "completed", "details": {"discovered": ["market_researcher","script_generator","content_generator","consistency_gate","platform_formatter","publisher","analytics_info_scalper"]}},
    {"step": "planning", "status": "completed", "details": {"central_plan": "..."}}
    ,
    {"step": "delegate_tasks", "status": "in_progress", "details": {"assignments": {
      "script_generator": {"depends_on": ["market_researcher"]},
      "content_generator": {"depends_on": ["script_generator"]},
      "consistency_gate": {"depends_on": ["content_generator"]},
      "platform_formatter": {"depends_on": ["consistency_gate"]},
      "publisher": {"depends_on": ["platform_formatter"]},
      "analytics_info_scalper": {"depends_on": ["publisher"]}
    }}}
  ],
  "media_asset": {
    "video_id": null,
    "video_urls": {},
    "publish_status": "pending"
  },
  "evidence": {
    "events": [],
    "communications": [],
    "state_transitions": []
  },
  "plan_history": [
    // persisted per-run history entries
  ],
  "logs": [
    // platform-native logs of all steps
  ],
  "next_commands": [
    "Wait for agents to complete their tasks; publish if all pass."
  ]
}
Examples:

| User Request | Tool Input |

|---|---|

| "Create today's daily marketing video for YouTube and Instagram about Platform Alpha." | {"target_date":"2026-07-07","platforms":"youtube,instagram","topic":"Platform Alpha launch","style":"brand-safe"} |

| "Prepare a daily onboarding video for new users on YouTube only." | {"target_date":"2026-07-07","platforms":"youtube","topic":"Onboarding for Platform Alpha","style":"educational"} |

| "Publish yesterday's video to both channels if ready." | {"target_date":"2026-07-06","platforms":"youtube,instagram"} |

| "Produce a short teaser for social channels about the latest platform feature." | {"target_date":"2026-07-07","platforms":"youtube,instagram","topic":"New feature teaser","style":"short-form"} |

| "Create a daily marketing video for Platform Alpha focusing on pricing update." | {"target_date":"2026-07-07","platforms":"youtube,instagram","topic":"Pricing update","style":"informative"} |

| "Generate a full daily video script and content plan for onboarding." | {"target_date":"2026-07-07","platforms":"youtube,instagram","topic":"Onboarding overview","style":"educational"} |

Detection heuristics — use this tool when the user mentions:

daily marketing workflows, content pipelines, or multi-agent orchestration
publishing on YouTube or Instagram
phrases like “plan end-to-end,” “delegate tasks,” “dynamic discovery,” “message bus,” or “centralized plan”
requests for autonomous operation, retries on failure, and alert/pause on persistent failure
requirements for daily cadence, persistence, and plan history
Safety Constraints
READ/WRITE: Mutating operations are allowed (create and publish media), but only within the defined workflow and platforms.
ROLE/Scope: Do not alter the predefined workflow topology (agent set, data flows, or orchestration graph). The brain may re-plan per run but must not rewire the overall graph or agent roster.
Data handling: No secret store usage; credentials and agent reachability are derived from Agent Cards published by each A2A agent and, if needed, per-agent tokens exposed in those cards. Do not print secrets in logs or outputs.
Egress: All external communications must go through permitted domains (as listed in the frontmatter). No connections to unlisted domains.
Tools: Denied tools include web_search and http_request; use the mandated binaries (curl, jq, etc.) and the provided REST endpoints for agent communication.
Autonomy limits: The brain retries on failure; if all retries fail, it must raise an alert and pause the workflow until human intervention confirms the issue.
Logging: All activity must be logged to the platform’s own logs; avoid exposing raw secrets or internal state externally.
Compliance: Only politically correct and morally aligned domains/topics may be chosen by the brain; content must comply with platform policies and advertising regulations.
Workflow immutability: Do not alter the core workflow topology or agent roster; re-planning per run is allowed, but the overall graph remains fixed.
Important Notes
Dynamic discovery: Agents publish their agent cards (A2A) at known endpoints; the brain reads these cards at runtime to assemble the active roster and their capabilities. No central secret store is required.
Secret management: Since there is no secret store, credentials are obtained via Agent Cards or per-agent environment configuration if provided by the A2A agents. Secrets must never be logged.
Plan history: Each daily run is recorded in a persistent plan_history store for auditing and continuous improvement.
Inter-agent messaging: A hybrid approach combines a message bus for asynchronous signal exchange and REST calls for direct requests to agents. The brain ensures proper sequencing and dependency management (task graph with centralized planning).
Daily cadence: The orchestrator is designed to run once per day; if the daily run fails to complete, retries occur automatically until success, after which the workflow resumes the normal cadence.
Localization and safety: All content generation adheres to platform safety guidelines, and all outputs are constrained to the allowed domains and formats.
Extensibility: The design supports adding or swapping agents without changing the core brain logic, via dynamic A2A agent cards and adapter-based communication.
