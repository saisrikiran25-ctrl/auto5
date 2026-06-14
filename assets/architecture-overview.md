# Architecture Overview

Loop Leak Detector Automation — Workflow Architecture Documentation

---

## System Summary

The Loop Leak Detector Automation is a multi-stage n8n workflow that combines deterministic event processing with AI-powered behavioral interpretation. It accepts a structured activity payload, identifies repeated behavioral patterns across digital objects, interprets them as attention leaks, and returns a structured diagnostic response.

The system is designed around a key principle: **not all AI output should be trusted unconditionally**. A deterministic layer runs before and after AI interpretation to validate clustering conditions and override AI output when objective signal is stronger.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        INPUT LAYER                                      │
│                                                                         │
│   External System / Manual Trigger                                      │
│          │                                                              │
│          ▼                                                              │
│   [ Webhook ] ──→ Accepts POST with activity payload                    │
└─────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    CONFIGURATION LAYER                                   │
│                                                                         │
│   [ Provider Config ]                                                   │
│   - Sets apiUrl, model, credentialName                                  │
│   - Sets clustering thresholds                                          │
│   - Injects _debug_provider_config into data pipeline                  │
└─────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    NORMALIZATION LAYER                                   │
│                                                                         │
│   [ Normalize Events ]                                                  │
│   - Validates required fields                                           │
│   - Normalizes event structure                                          │
│   - Sorts events chronologically                                        │
│   - Throws on missing required fields                                   │
└─────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    CLUSTERING LAYER                                      │
│                                                                         │
│   [ Cluster Repeated Behavior ]                                         │
│   - Groups events by object_name + object_category                     │
│   - Counts revisits, edits, reopens, defers, sends, submits            │
│   - Tracks unique sessions per cluster                                  │
│   - Selects primary cluster by highest event volume                    │
│   - Builds event_timeline array                                         │
└─────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    VALIDATION GATE                                       │
│                                                                         │
│   [ Validate Cluster ]                                                  │
│   - Checks cluster against configured thresholds                       │
│   - Sets cluster_is_valid: true/false                                  │
│   - Sets cluster_invalid_reason if false                               │
│                                                                         │
│   [ If Valid ] ────── false ──→ [ Low Signal Response ]                │
│        │                              │                                 │
│        │ true                         ▼                                 │
│        │                    Returns low_signal_noise                    │
│        │                    without calling AI                         │
└─────────────────────────────────────────────────────────────────────────┘
                          │ (valid path)
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    AI INTERPRETATION LAYER                               │
│                                                                         │
│   [ Prep Loop Analysis Request ]                                        │
│   - Constructs system + user prompt messages                           │
│   - Interpolates cluster data into user prompt                         │
│   - Builds HTTP request body for AI provider                           │
│                                                                         │
│   [ AI: Analyze Loop ]                                                  │
│   - HTTP POST to configured AI provider endpoint                       │
│   - Uses HTTP Header Auth credential from n8n vault                    │
│   - Retries up to 3 times on failure                                   │
│                                                                         │
│   [ Parse Loop Analysis ]                                               │
│   - Extracts content from AI response                                  │
│   - Strips code fences if present                                      │
│   - Validates required output fields                                   │
│   - Falls back gracefully on parse failure                             │
└─────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    DETERMINISTIC VALIDATION LAYER                        │
│                                                                         │
│   [ Deterministic Friction Checks ]                                     │
│   - Rule 1: Escalate severity on very high revisit count + no closure  │
│   - Rule 2: Override loop_type to draft_churn_loop on high edit + zero send │
│   - Rule 3: Flag defer_reopen_cycle candidate if both counts exceeded  │
│   - Rule 4: Downgrade confidence if total_events < 4                   │
│   - Rule 5: Set needs_human_review on critical severity                │
│   - Appends deterministic flags to _debug_ field                       │
└─────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    REVIEW ROUTING LAYER                                  │
│                                                                         │
│   [ Human Review Override ]                                             │
│   - Reads needs_human_review flag                                      │
│   - Sets _debug_human_review_pending                                   │
│   - In production: connect a notification node here                    │
│                                                                         │
│   [ If Weekly Review ] ── analysis_mode == weekly_review? ──┐          │
│        │                                                     │          │
│        │ no                                                  │ yes      │
│        ▼                                                     ▼          │
│   (straight to Final Response)               [ Prep Weekly Pattern Review ] │
│                                              [ AI: Weekly Pattern Review ]   │
│                                              [ Parse Weekly Pattern Review ] │
└─────────────────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    OUTPUT LAYER                                          │
│                                                                         │
│   [ Final Response ]                                                    │
│   - Assembles clean public response object                             │
│   - Strips ALL _debug_ prefixed fields                                 │
│   - Merges loop_analysis and optional weekly_review                    │
│   - Returns only fields defined in output schema                       │
│                                                                         │
│   [ Respond to Webhook ]                                               │
│   - Returns HTTP 200 with clean JSON body                              │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Stage Descriptions

### Stage 1: Input Capture

The automation is triggered by a POST request to the webhook endpoint. The webhook accepts any valid JSON body and passes it downstream. No authentication is applied at the webhook level by default — add n8n webhook authentication if you need to restrict access.

### Stage 2: Configuration Injection

The Provider Config node is the single source of truth for AI provider settings and clustering thresholds. It injects configuration into the data pipeline as `_debug_provider_config` so all downstream nodes can access settings without external calls. All user-facing customization happens here first.

### Stage 3: Event Normalization

The Normalize Events node enforces the input contract. It validates required fields, normalizes string casing and whitespace, and sorts events chronologically. Any malformed or incomplete payload is rejected here with a descriptive error before reaching any processing logic.

### Stage 4: Repeated Behavior Clustering

The Cluster Repeated Behavior node groups events by `object_name + object_category` and computes behavioral counters per cluster. This is entirely deterministic. No AI is involved. The primary cluster (highest event volume) is selected for analysis.

### Stage 5: Validation Gate

The Validate Cluster node applies threshold checks to the primary cluster. If the cluster is too small or does not meet any loop-type minimum, it is classified as low-signal and the workflow short-circuits to a structured response without calling the AI provider. This prevents API cost and over-interpretation of noise.

### Stage 6: AI Interpretation

The AI layer consists of three nodes: prompt construction, HTTP request, and response parsing. The Prep node builds structured messages from the cluster data. The AI node sends the request to the configured provider with retry logic. The Parse node extracts and validates the JSON response, with graceful fallback for malformed output.

### Stage 7: Deterministic Friction Checks

After AI interpretation, five rule-based checks run to validate and optionally override the AI output. These rules apply objective behavioral logic that does not require model inference — for example, if `draft_edit_count >= 3` and `send_count == 0`, the loop type is definitionally `draft_churn_loop` regardless of what the model classified it as. This layer prevents AI hallucinations from propagating into the final response.

### Stage 8: Review Routing

The Human Review Override node checks whether the result is flagged for review. In the base configuration, it passes through — but is designed as the insertion point for notification logic (Slack, email, etc.). The If Weekly Review node then routes to the optional second-pass AI call if `analysis_mode == weekly_review`.

### Stage 9: Clean Response Generation

The Final Response node assembles the public output by explicitly mapping result fields into a clean object. The `_debug_` prefix convention ensures that all internal fields are easily excluded — the Final Response node only assigns fields defined in the output schema. No debug data leaks into the public response.

---

## Data Flow Conventions

### `_debug_` Prefix

All internal and diagnostic fields use the `_debug_` prefix:
- `_debug_provider_config` — configuration object
- `_debug_stage` — current workflow stage name
- `_debug_ai_request_body` — constructed AI request
- `_debug_ai_raw_response` — raw AI provider response
- `_debug_parse_error` — parse failure message if applicable
- `_debug_deterministic_flags` — list of deterministic override messages
- `_debug_human_review_pending` — boolean review flag

All `_debug_` fields are stripped in the Final Response node.

### `_loop_analysis_result` and `_weekly_review_result`

Intermediate result objects are stored with single underscore prefix to distinguish them from debug fields. The Final Response node maps these into the clean public structure.

---

## Design Decisions

**Why deterministic checks after AI, not before?**
The AI model may identify patterns that deterministic rules cannot — for example, recognizing that a `research_spiral` involves many related objects. The deterministic layer validates and corrects AI output rather than replacing it.

**Why a separate cluster validation gate?**
Calling the AI provider on low-signal noise is wasteful and produces unreliable output. The gate prevents both cost and over-interpretation.

**Why is `_debug_provider_config` in the data pipeline instead of a separate store?**
n8n's code nodes do not have native shared state across nodes. Injecting config into the pipeline is the most portable approach that works across all n8n versions without requiring external storage.

**Why strip `_debug_` fields only at the Final Response stage?**
Keeping debug fields throughout the pipeline makes node-level debugging straightforward in the n8n execution log. Stripping only at the end means you can inspect intermediate state without modifying any upstream node.
