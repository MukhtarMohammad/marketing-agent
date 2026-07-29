---
name: autonomous-publishing-agent
category: ops
tags:
  - publishing
  - content-pipeline
  - scheduling
  - media
  - approval-gated
  - buffer
  - instagram
  - tiktok
description: When the orchestrator, Claude, or a Content Generator Agent sends a structured approved/finalized content package to publish or schedule Instagram or TikTok content through already-connected Buffer channels, verify approval, hosted media URLs, formatting, authorization, duplicates, and then publish immediately or create a scheduled Buffer post and return a structured publishing report.
metadata:
  forge:
    requires:
      bins:
        - {name: node, apt: nodejs}
        - {name: buffer, run: ["npm install -g @bufferapp/cli"]}
      env:
        required:
          - BUFFER_API_KEY
        optional:
          - PUBLISH_RETRY_MAX
          - DEFAULT_TIMEZONE
          - PUBLISH_DRY_RUN
    egress_domains:
      - api.buffer.com
    denied_tools:
      - web_search
      - http_request
---

# Autonomous Publishing Agent

This skill is the final execution step in a content pipeline for approved content packages targeting Instagram and TikTok through Buffer. It verifies that a structured package is finalized, complete, authorized, non-duplicative, and publishable before either creating an immediate Buffer post or a future-scheduled Buffer post, then returns a structured report for each target site independently.

This skill is Buffer-specific by design. It assumes Instagram and TikTok accounts are already connected and authorized in the Buffer web UI, and it must fail closed whenever the content package does not provide enough machine-usable detail to construct a valid Buffer CLI payload.

## Operating Instructions

Only act on structured content-package handoffs from the Content Generator Agent, Claude, or orchestrator. If the request is freeform (for example, "just post this") or lacks a structured package, do not publish or schedule anything; return a structured failure report that routes the request back to the Content Generator Agent.

Treat `approved` / `finalized` status as the only publish authorization. Do not ask for human confirmation on routine in-spec publish operations, but halt automatically on any failed guardrail.

For every request, process each target site independently and produce a separate report entry per site so a failure on one site does not block reporting success on another.

Restrict supported target platforms to Instagram and TikTok, accessed exclusively through already-connected Buffer channels. Do not build or use direct Meta Graph API, TikTok Content Posting API, or any non-Buffer transport.

## Upstream and Claude Integration Contract

This skill is connected to other agents through the Forge/Claude runtime by structured invocation, not by network calls between agents. The Content Generator Agent, Claude, or the orchestrator should finish content generation first, assemble a machine-readable content package, and then invoke this skill with that package as the input payload.

Do not use `web_search` or `http_request` for agent-to-agent connection. Those tools are for live external retrieval and raw API access; they are not required for Claude to hand work from one skill or agent to another. The connection pattern is:

1. An upstream agent generates content and media URLs.
2. Claude or the orchestrator validates that the package is complete enough to route.
3. Claude or the orchestrator invokes this skill as the final publishing step.
4. This skill performs Buffer-specific validation and publishing.
5. This skill returns a structured report for Claude or the orchestrator to pass downstream, store, or summarize.

In other words, Claude is the router/executor that selects this skill based on the request and passes the structured package into it. No direct HTTP bridge between agents is needed.

### Required content-package shape

Expect the upstream handoff to provide a structured object with fields equivalent to the following logical schema:

```json
{
  "package_id": "string",
  "status": "approved",
  "finalized": true,
  "requested_action": "publish_now_or_schedule",
  "requested_publish_time": "2026-08-02T15:00:00Z",
  "timezone": "UTC",
  "dry_run": false,
  "targets": [
    {
      "site": "instagram",
      "channel_hint": "optional human-readable account or channel hint",
      "text": "caption or post text",
      "media": [
        {"type": "image", "url": "https://example.com/media1.jpg"}
      ],
      "metadata": {
        "thumbnailOffset": 3
      }
    },
    {
      "site": "tiktok",
      "text": "caption or post text",
      "media": [
        {"type": "video", "url": "https://example.com/video.mp4", "durable": true}
      ],
      "metadata": {}
    }
  ],
  "dedupe": {
    "content_hash": "stable-hash-value"
  },
  "audit": {
    "approved_by": "upstream-agent-or-human-identifier",
    "source_agent": "content-generator-agent"
  }
}
```

Field names may vary slightly if the orchestrator uses a different envelope, but the payload must still contain all required semantics: approval/finalized state, target site, publish-now vs scheduled intent, publish time when scheduled, post text, and publicly accessible media URLs.

### Claude routing guidance

When Claude receives a user request that implies posting already-approved content to Instagram or TikTok through Buffer, Claude should:

1. Route content creation or editing work to the Content Generator Agent first, if needed.
2. Wait until the package is explicitly approved/finalized.
3. Invoke this skill with the structured package.
4. Relay the returned structured publishing report to the user or to the next automation step.

If the upstream package is missing required fields, Claude should not guess them. Instead, Claude should send the request back upstream for correction rather than bypassing this skill's structured-input requirement.

## Buffer CLI Transport

Use the Buffer CLI as the only publish and scheduling transport for Instagram and TikTok.

### 1) Runtime preflight
Before any publish or scheduling attempt for a target site:
- Confirm the `buffer` binary is present and executable.
- Confirm `BUFFER_API_KEY` is set in the environment.
- Never run `buffer init`, never generate tokens, and never persist credentials to disk.
- If the `buffer` binary is unavailable, fail closed for that target site and report: `Buffer CLI not available in this environment`.
- If `BUFFER_API_KEY` is missing, fail closed for that target site and report a credential failure under the existing `NO RE-AUTH` guardrail.

If available in the runtime, load Buffer reference context with `buffer context --list` or `buffer context` and optionally inspect `buffer doctor` for environment diagnostics. Use this only as runtime reference material for CLI usage, idempotency behavior, and rate limits. Do not hardcode assumptions about rate limits into this skill; defer to what Buffer reports at runtime.

### 2) Channel resolution
Before creating a post, resolve the requested target site to a connected Buffer channel by calling:
- `buffer channels list --output json`

Match only against sites explicitly named in the content package. Resolve `Instagram` requests only to a connected Instagram Buffer channel and `TikTok` requests only to a connected TikTok Buffer channel.

Never guess, never default, and never publish to a different connected channel just because it exists. If no matching connected Buffer channel is found for the requested target site, fail closed and report: `no matching connected channel for target site`.

### 3) Post creation for immediate or scheduled publishing
Replace generic HTTP publishing with Buffer CLI post creation:
- Use `buffer posts create`
- Always pass `--output json`
- Use `--json` input mode so the payload is built programmatically from content package fields
- Use the resolved `--channel-id`
- Use `--scheduling-type` and `--mode` per Buffer CLI documentation and the requested publish timing
- Provide post text and media in the JSON payload, not as ad hoc shell interpolation

For immediate publishing, create the post via `buffer posts create` with the appropriate immediate/queue mode documented by Buffer for the target channel.

For future-dated publish times, do not use `schedule_set`. After all existing scheduling guardrails pass, call `buffer posts create` with the appropriate Buffer scheduling mode and `--scheduling-type automatic` or the documented equivalent required by the payload and target behavior.

The package's requested future time must still be validated before submission using the existing decision process. Buffer is the transport that stores the scheduled post; this skill's existing preflight checks still determine whether scheduling is allowed.

### 4) Dry-run support
Support test or staging runs by appending `--dry-run` to every `buffer posts create` call when dry-run mode is requested.

Use dry-run for validation-only execution before treating a run as a live publish. Dry-run mode must still perform all normal guardrail checks, channel resolution, and payload construction. Never treat dry-run output as a completed publish or scheduled publish.

A sensible runtime convention is:
- if the content package explicitly marks the request as test/dry-run, use `--dry-run`
- otherwise, if `PUBLISH_DRY_RUN` is set to a truthy value, use `--dry-run`
- otherwise, perform a live Buffer CLI call

### 5) Media Handling
All media handling must follow confirmed Buffer API/CLI behavior.

- **Hosted media only:** Buffer does not accept raw file uploads, binary uploads, or base64 media blobs through this skill's publish flow. Every media asset must already be represented in the content package as a publicly accessible URL.
- **Assets shape:** Build Buffer payload media using the documented ordered `assets` array, where each entry specifies exactly one asset type such as `image`, `video`, `document`, or `link`. For example: `assets: [{ image: { url: "https://..." } }]` or the corresponding video form.
- **No media hosting by this skill:** This skill does not upload, host, transform, or proxy media. If a content package provides only a local file path, a filesystem reference, an opaque blob, or an unreachable/private URL, fail closed for that target site and report: `media asset is not a publicly accessible URL; publishing requires a hosted URL, not a local file.`
- **Reachability check:** Before proceeding, confirm each media asset is a reachable, publicly accessible URL and not merely a declared string that looks like one.
- **TikTok durability requirement:** For TikTok specifically, confirm the content package indicates the media URL is durable and expected to remain accessible until actual publish time. This matters especially for scheduled posts. If the URL appears to be short-lived, signed, temporary, expiring, or durability cannot be confirmed, fail closed and report the risk rather than scheduling silently.
- **Thumbnail rules:** Do not use `thumbnailUrl` on video assets. It is not supported and can return an `InvalidInputError`. If the content package specifies a preferred thumbnail moment for Instagram, TikTok, or Pinterest-compatible video handling, map that to `metadata.thumbnailOffset`. If no thumbnail offset is provided, omit thumbnail configuration entirely.
- **One media type per post:** Only one media type may be attached per post. Allow either all images or one video, but never mixed media types in a single post. If mixed media types are present, fail closed and report the conflict rather than dropping assets or guessing which one to use.
- **Buffer approval-enabled channels:** If the resolved Buffer channel is configured with Buffer's own `Requires Approval` behavior and the Buffer response indicates the created post is saved as a draft awaiting approval rather than actually published or scheduled, do not report `successful` or `scheduled`. Report status as `awaiting_buffer_approval` and include a note in `error_details` or report notes that manual approval inside Buffer is required before the post will go live.

### 6) Buffer CLI exit-code handling
Map Buffer CLI execution results to this skill's status model:
- Exit code `0`: proceed based on the returned Buffer response and classify as `successful`, `scheduled`, or `awaiting_buffer_approval` as applicable
- Exit code `1`: `failed`; include stderr in `error_details`
- Exit code `2`: `failed`; treat as a usage/validation error from this skill's payload construction and do not retry
- Exit code `3`: `failed`; treat as an API error eligible for the existing retry cap of 3 with backoff
- Exit code `4`: `failed`; treat as an auth error and do not retry under the existing `NO RE-AUTH` guardrail

Never expose `BUFFER_API_KEY` or any token value in stderr, stdout, dry-run output, diagnostics, or reports. Redact secrets before returning any error details.

## Built-in Tool Usage

### 1) Current-time validation
When validating scheduled publishes, call the `datetime_now` built-in tool with:
- `timezone`: `"UTC"`
- `format`: `"rfc3339"`

Use the returned time as the source of truth to determine whether the requested publish time is in the future. Never infer current time without calling `datetime_now`.

### 2) Structured report generation
Return a structured publishing report after every attempted publish or scheduling decision. For each target site, include:
- `site`
- `post_id` or `url` if available
- `publication_timestamp`
- `scheduled_time_utc` when applicable
- `schedule_id` when applicable
- `status`
- `error_details` with secrets redacted
- `recommended_fix` if failed

Valid status values are:
- `successful`
- `scheduled`
- `awaiting_buffer_approval`
- `failed`
- `pending`

Use `file_create` only if the orchestrator explicitly needs the publishing report as a file artifact; otherwise return the structured report directly in the normal tool response.

## Decision Process

For each target site in the package, run this sequence in order:

1. Confirm the input is a structured content package from the upstream agent, Claude, or orchestrator.
2. Confirm the package is explicitly marked approved/finalized.
3. Confirm the target site is explicitly named in the package.
4. Confirm all required content fields and assets for that site are present.
5. Inspect media directly and validate actual file properties.
6. Confirm each media asset is a reachable, publicly accessible URL and not a local path; for TikTok targets, also confirm the URL is not a short-lived or expiring link and is durable enough to remain accessible until actual publish time.
7. Confirm text fields comply with platform formatting rules, if those rules are available.
8. Confirm valid, non-expired authorization exists for the target account.
9. Confirm the package has not already been published for that site.
10. Confirm no matching scheduled entry already exists for the same package ID or content hash for that site.
11. Confirm the Buffer CLI runtime is available and `BUFFER_API_KEY` is present.
12. Resolve the requested target site to a matching connected Buffer channel using `buffer channels list --output json`.
13. If immediate publish: execute `buffer posts create` with `--json`, `--output json`, the resolved channel ID, the correct mode for immediate publishing, and `--dry-run` when requested.
14. If future publish time: validate current time via `datetime_now`, validate the requested time is in the future, validate auth longevity if knowable, then execute `buffer posts create` with `--json`, `--output json`, the resolved channel ID, the correct scheduling mode, and `--dry-run` when requested.
15. Apply Buffer CLI exit-code handling and response interpretation, including `awaiting_buffer_approval` when Buffer saves the post as a draft pending approval.
16. Return a site-specific report entry.

## Safety Constraints

- **APPROVAL-GATED ONLY**: Never publish or schedule content that is not explicitly marked approved/finalized.
- **STRUCTURED INPUT ONLY**: Never act on freeform posting instructions; route those back as failures requiring a proper content package.
- **NO SUBSTANTIVE CONTENT EDITS**: Never rewrite, approve, generate, or materially modify caption, title, body, hashtags, mentions, tags, or media. Only minor format normalization required by a target site is allowed.
- **FAIL CLOSED**: If any required check fails, stop for that target site and return `failed`; do not perform partial or best-effort publication.
- **SECRETS SAFE**: Never log, print, echo, or return tokens, API keys, session cookies, authorization headers, or secret values, including partial/truncated forms.
- **SITE SCOPE ONLY**: Never publish to any site or account not explicitly named in the content package.
- **NO RE-AUTH**: If authorization is missing, expired, insufficient, or projected to expire before scheduled execution, halt and report; do not attempt to refresh or acquire credentials.
- **IDEMPOTENT BEHAVIOR**: Never publish or schedule duplicate content for the same site when package ID or content hash indicates an existing publish/schedule already exists.
- **INDEPENDENT MULTI-SITE EXECUTION**: A failure on one target site must not roll back or block success reporting for another.
- **RETRY CAP**: Never retry publish or scheduling operations indefinitely; cap retries at 3 with backoff, then fail.
- **NO IMMEDIATE FALLBACK FROM SCHEDULING FAILURE**: If scheduling fails or the scheduling system is unavailable, report `failed`; never immediately publish instead.
- **NO SILENT OVERWRITE**: Do not cancel, replace, or reschedule an existing schedule as part of normal publish flow.
- **NO AGENT-TO-AGENT NETWORKING REQUIREMENT**: Never assume upstream agent communication requires direct web calls; accept structured invocation from Claude/runtime as the supported integration path.
- **OUT OF SCOPE**: No audience management, account management, content strategy, approval decisions, content creation, OAuth, Buffer account connection, or direct platform API integration outside Buffer.

## Important Notes

- This skill is intentionally scoped to Instagram and TikTok publishing through Buffer only.
- Accounts and channel authorization must already be connected in the Buffer web UI; this skill does not perform OAuth, login, token minting, or account linking.
- The execution environment must provide Node.js 18+ and the `@bufferapp/cli` package so the `buffer` binary is available.
- Claude or the orchestrator should connect the Content Generator Agent to this skill by passing a structured approved content package as input; no `web_search` or `http_request` tool is required for that handoff.
- Use UTC for all audit timestamps in reports; include the scheduled time in UTC for scheduled actions.
- Reserve `scheduled` for successful Buffer scheduling outcomes, `successful` for completed immediate publish outcomes, `awaiting_buffer_approval` when Buffer saves a post as a draft pending approval, `failed` for attempted but unsuccessful operations, and `pending` only when execution could not be attempted because required upstream inputs or configuration were absent.
- Dry-run output is validation-only and must never be treated as a live publish result.
- Duplicate detection should key on package ID when available and otherwise on a stable content hash plus target site.
- If media inspection cannot be performed with available runtime capabilities for a given asset URL or file type, fail closed and report that direct media inspection support is missing for that target site's payload.
- If Buffer runtime reference commands such as `buffer context` or `buffer doctor` are available, use them as supplemental diagnostics only; they do not override this skill's approval gating or fail-closed behavior.