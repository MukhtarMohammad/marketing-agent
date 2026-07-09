---
name: google-veo-marketing-videos
category: marketing
tags:
  - ai-video
  - video-generation
  - marketing
  - automation
  - veo
description: Generate 15-second marketing videos by calling Google VEo API using a provided prompt
metadata:
  forge:
    requires:
      bins:
        - curl
        - jq
        - gcloud
    env:
      required:
        - VEO_API_KEY
    egress_domains:
      - veo.googleapis.com
      - googleapis.com
      - storage.googleapis.com
    denied_tools:
      - http_request
    timeout_hint: 10800
---
# Google VEo Marketing Video Generator

This skill creates 15-second marketing videos by invoking the Google VEo API using a prompt provided by another agent. It relies on a VEo API key supplied at runtime via the VEO_API_KEY environment variable and uses curl (with optional gcloud support) to perform the API calls. The output is a video asset delivered via a URL, along with a job/status payload for tracking and auditing. Content is restricted to strictly informational, non-political, non-adult, and non-abrasive material.

## Tool: google_veo_video

Generate a 15-second (configurable) marketing video from an external prompt using Google VEo API.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| prompt | string | Yes | The creative prompt or brief provided by the other agent to generate the video |
| duration_seconds | integer | No | Video length in seconds (default: 15). Must be 15 per policy unless overridden by the prompt; recommended to fix at 15. |
| language | string | No | Language/callback locale for narration and captions (e.g., "en-US"). Default if omitted. |
| voice_profile | string | No | VEo voice profile or ID to use for narration (e.g., "default", "female_en_US"). |
| platform | string | No | Intended distribution platform (e.g., "YouTube", "Instagram", "TikTok"). |
| aspect_ratio | string | No | Output aspect ratio: "16:9", "9:16", or "1:1" (default: "16:9"). |
| brand_assets | object | No | Brand assets to embed: { "logo_url": "...", "palette": ["#RRGGBB", "..."] } |
| output_format | string | No | Video format for delivery (default: "mp4"). |
| notes | string | No | Additional notes or constraints for video generation (e.g., "avoid violent imagery"). |

**Output:**

```json
{
  "video_url": "https://storage.googleapis.com/your-bucket/path/to/video.mp4",
  "job_id": "veo-job-abcdef123",
  "status": "queued" | "in_progress" | "completed" | "failed",
  "duration_seconds": 15,
  "platform": "YouTube",
  "aspect_ratio": "16:9",
  "language": "en-US",
  "voice_profile": "default",
  "errors": null
}
```

**Notes on API key handling**

- The VEo API key must be provided at runtime via the environment variable VEO_API_KEY.
- The key must not be embedded in the SKILL.md or tool inputs and must not be logged.
- The curl call should include the key in the Authorization header (or as required by VEo docs), e.g. Authorization: Bearer $VEO_API_KEY.

**Examples:**

| User Request | Tool Input |
|---|---|
| "Create a 15-second YouTube promo for our new running shoes in English with a blue color palette." | {"prompt":"Create a 15-second YouTube promo for our new running shoes focusing on speed and comfort.","duration_seconds":15,"platform":"YouTube","aspect_ratio":"16:9","language":"en-US","voice_profile":"default","brand_assets":{"logo_url":"https://example.com/logo.png","palette":["#1D4ED8","#93C5FD"]}} |
| "Produce a 15-second Instagram teaser in English about eco-friendly bags." | {"prompt":"15-second Instagram teaser highlighting eco-friendly bags.","duration_seconds":15,"platform":"Instagram","aspect_ratio":"9:16","language":"en-US","brand_assets":{"logo_url":"https://example.com/logo.png","palette":["#34D399","#065F46"]}} |
| "15s TikTok ad for a coffee brand, Spanish language." | {"prompt":"15-second TikTok ad for a premium coffee brand with a bold, energetic vibe.","duration_seconds":15,"platform":"TikTok","aspect_ratio":"9:16","language":"es-ES","voice_profile":"default_es","brand_assets":{"logo_url":"https://example.com/logo.png","palette":["#6B3E90","#FFD166"]}} |
| "Explain the new data privacy policy in a 15-second video" | {"prompt":"A concise, factual 15-second explainer about our new data privacy policy.","duration_seconds":15,"platform":"YouTube","aspect_ratio":"16:9","language":"en-US"} |
| "15-second informational video about a product update for enterprise customers" | {"prompt":"15-second informational video about the latest product update aimed at enterprise customers.","duration_seconds":15,"platform":"YouTube","aspect_ratio":"16:9","language":"en-US"} |
| "15-second teaser for a mobile app launch, showing brand colors" | {"prompt":"15-second teaser for a mobile app launch highlighting brand colors and logo fade-in.","duration_seconds":15,"platform":"Instagram","aspect_ratio":"9:16","language":"en-US","brand_assets":{"logo_url":"https://example.com/logo.png","palette":["#0EA5E9","#1D4ED8"]}} |

**Detection heuristics** — Use this tool when the user mentions:
- phrases like “create a video,” “promo video,” “video ad,” “marketing video”
- references to 15 seconds, short-form videos, or platform-specific formats (YouTube, Instagram, TikTok, Shorts, Reels)
- requests to generate AI-driven marketing or promotional content from a prompt

## Safety Constraints

- No political content, no adult content, no abrasive or hate-based messaging
- Read-only for any existing data; this tool mutates by creating new video assets via VEo (mutating operation)
- Do not access or log sensitive credentials; do not expose the VEo API key in outputs
- Egress is restricted to the domains declared in the frontmatter:
  - veo.googleapis.com
  - googleapis.com
  - storage.googleapis.com
- Denied tools: http_request (avoid ad-hoc HTTP via non-curled methods)
- Do not use any domain outside the allowed egress list
- Do not generate content that could violate platform guidelines (e.g., misleading claims)

## Important Notes

- API key handling: The VEo API key must be supplied via the VEO_API_KEY environment variable. Do not log or echo the key.
- The script uses curl to call VEo endpoints and jq to parse/structure responses. It relies on gcloud only as a supporting binary; no direct OAuth token harvesting is required when using the API key approach.
- The 15-second requirement is enforced by default. If a different duration is needed, the prompt should explicitly request it, but the recommended default remains 15 seconds.
- Timeout: The skill assumes a maximum overall operation time of 3 hours (10800 seconds). If VEo takes longer, the agent should report a timeout and retry according to policy.
- Output includes a final video URL and a job_id/status, enabling downstream agents to poll for completion or failure.