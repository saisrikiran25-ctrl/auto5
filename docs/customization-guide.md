# Customization Guide

Loop Leak Detector Automation — Customization Reference

---

## Overview

The Loop Leak Detector is designed to be customized without modifying core workflow logic. Most customizations are made in one of three places:

1. **Provider Config node** — for provider, model, and threshold changes
2. **Prompt files** — for vocabulary, tone, and AI behavior changes
3. **Code nodes** — for deterministic logic, output structure, and routing changes

This guide covers all supported customization points in order of frequency of use.

---

## 0. Critical: Prompt File Synchronization

> **This is the most important customization warning in this guide. Read it before editing any prompt.**

The `prompts/` directory contains the canonical prompt files:
- `prompts/system-loop-analysis.md`
- `prompts/user-loop-analysis-template.md`
- `prompts/system-weekly-pattern-review.md`
- `prompts/user-weekly-pattern-review-template.md`

**However, the workflow nodes do NOT load these files at runtime.**

The `Prep Loop Analysis Request` and `Prep Weekly Pattern Review` nodes contain **inline copies** of the prompt text embedded directly as string literals in their JavaScript code. This is because n8n Code nodes do not have native access to the local filesystem at execution time.

This creates a **divergence risk**: if you edit the `.md` file but not the matching inline string in the Code node (or vice versa), the files and the running workflow will be out of sync. The `.md` files serve as the source-of-truth documentation, but the Code node strings are what actually execute.

### Rule: Always Update Both

Whenever you change a system or user prompt:

1. Edit the `.md` file in `prompts/` first (this is your source of truth).
2. Copy the updated text into the matching Code node's inline string in n8n.
3. Save the Code node.
4. Test with a sample payload to confirm the change is live.

### How to Load Prompt Files Dynamically (Advanced)

If you want to avoid maintaining duplicate copies, you have two options in n8n:

**Option A — n8n Read/Write Files node (self-hosted only):**
Insert an `n8n-nodes-base.readBinaryFiles` node before the Prep node. Configure it to read `prompts/system-loop-analysis.md` from the n8n working directory. Then extract the file content in the Prep node:
```javascript
const systemPrompt = $('Read Prompt File').first().binary.data.toString('utf-8');
```
This only works on self-hosted n8n instances with filesystem access enabled.

**Option B — Environment variable injection:**
Store the prompt text as an environment variable in your n8n instance (e.g., `LOOP_ANALYSIS_SYSTEM_PROMPT`). Access it in the Code node:
```javascript
const systemPrompt = process.env.LOOP_ANALYSIS_SYSTEM_PROMPT || 'fallback prompt text';
```
This is the most portable approach and works on both self-hosted and n8n Cloud.

Until you implement one of the above, treat the inline string in the Code node as the live prompt and keep the `.md` file in sync manually.

---

## 1. Changing the AI Provider or Model

**Where:** `Provider Config` node

Update these fields:

```javascript
const config = {
  apiUrl: 'https://api.openai.com/v1/chat/completions',  // ← change endpoint
  model: 'gpt-4o',                                        // ← change model
  credentialName: 'YOUR_CREDENTIAL_NAME_HERE',            // ← change credential
  maxTokens: 1024,                                        // ← adjust if needed
  temperature: 0.2                                        // ← adjust for variability
};
```

Compatible endpoints (OpenAI-format):
- OpenAI: `https://api.openai.com/v1/chat/completions`
- Groq: `https://api.groq.com/openai/v1/chat/completions`
- Mistral: `https://api.mistral.ai/v1/chat/completions`
- Together AI: `https://api.together.xyz/v1/chat/completions`
- Ollama (local): `http://localhost:11434/v1/chat/completions`

See [`docs/credential-setup.md`](credential-setup.md) for credential switching instructions.

---

## 2. Adjusting Clustering Thresholds

**Where:** `Provider Config` node → `thresholds` object

These values control when a behavioral cluster is considered significant enough to analyze:

```javascript
thresholds: {
  minRevisitCount: 3,       // Minimum open/view events to qualify as a revisit loop
  minDraftEdits: 3,         // Minimum edit events to qualify as a draft churn loop
  minReopenCount: 2,        // Minimum explicit reopen events for reopen loop detection
  minDeferCount: 2,         // Minimum defer events for defer_reopen_cycle detection
  lowSignalEventCount: 2    // Events at or below this count are classified as low signal
}
```

**Increase thresholds** to reduce false positives (only flag high-frequency loops).
**Decrease thresholds** to catch lower-frequency patterns (useful for shorter time windows or sparse data).

Recommended starting points by use case:
| Use case | minRevisitCount | minDraftEdits | lowSignalEventCount |
|---|---|---|---|
| Daily analysis (tight window) | 2 | 2 | 1 |
| Weekly analysis (default) | 3 | 3 | 2 |
| Monthly analysis (wide window) | 5 | 5 | 3 |

---

## 3. Adjusting Revisit Sensitivity

**Where:** `Cluster Repeated Behavior` node

The cluster node counts multiple event types as "revisits":

```javascript
revisit_count: (etc['tab_open'] || 0) + (etc['item_open'] || 0) + 
               (etc['file_open'] || 0) + (etc['reopen'] || 0),
```

To add additional event types to the revisit count, extend this expression:

```javascript
revisit_count: (etc['tab_open'] || 0) + (etc['item_open'] || 0) + 
               (etc['file_open'] || 0) + (etc['reopen'] || 0) +
               (etc['view'] || 0) + (etc['preview'] || 0),  // ← add your event types
```

To make the automation more sensitive to quick re-opens within the same session, add session-level counting logic in the cluster node.

---

## 4. Adjusting Draft Churn Thresholds

**Where:** `Provider Config` node (threshold) + `Deterministic Friction Checks` node (override logic)

The draft churn override rule in `Deterministic Friction Checks`:

```javascript
// Rule 2: Force loop type to draft_churn_loop if draft_edit_count is high and send_count is zero
if (cluster.draft_edit_count >= thresholds.minDraftEdits && cluster.send_count === 0) {
  result.loop_type = 'draft_churn_loop';
}
```

To also trigger on objects with a non-zero send count but still high edit count:

```javascript
if (cluster.draft_edit_count >= thresholds.minDraftEdits && 
    cluster.draft_edit_count > cluster.send_count * 3) {
  // More than 3x edits per send = likely draft churn
  result.loop_type = 'draft_churn_loop';
}
```

---

## 5. Customizing Friction Labels

**Where:** `prompts/system-loop-analysis.md`

The system prompt defines the valid enumerated values for `friction_category`. To add a custom friction category:

1. Add the new value to the enumeration list in the system prompt under **Friction Category Classification**.
2. Add it to the `friction_category` field's `enum` array in `schemas/loop-analysis-output-schema.json`.

Example: Adding `social_dependency` as a friction category:
```
- `social_dependency` — The object cannot be resolved without input or action from another person.
```

Keep the list short. More than 10 friction categories will degrade AI classification quality.

---

## 6. Customizing Intervention Wording Style

**Where:** `prompts/system-loop-analysis.md` → tone instructions

The system prompt includes explicit anti-patterns for intervention language. To shift the style:

**To make interventions more direct:**
Add to the system prompt:
```
Intervention language must be imperative. Use action verbs. No hedging.
```

**To make interventions slightly more exploratory:**
Add to the system prompt:
```
Intervention language may include a brief rationale for why the specific action addresses the identified blocker.
```

Do not remove the core constraint against generic productivity advice.

---

## 7. Adjusting Output Tone by Request

**Where:** Input payload → `preferred_tone` field

The `preferred_tone` field is passed to the user prompt and instructs the model to adjust phrasing:
- `direct` — Minimal hedging. Imperative interventions. (default)
- `neutral` — Balanced phrasing.
- `reflective` — Slightly more exploratory framing in recommended_next_step.

The JSON structure does not change based on tone.

---

## 8. Adding Custom Loop Types

**Where:** `prompts/system-loop-analysis.md` + `schemas/loop-analysis-output-schema.json`

To add a new loop type (e.g., `approval_waiting_loop`):

1. Add the new type to the **Loop Type Classification** section in `system-loop-analysis.md`:
```
- `approval_waiting_loop` — Object is revisited repeatedly while awaiting external approval. No self-directed action is possible until approval arrives.
```

2. Add the value to the `loop_type` enum in `loop-analysis-output-schema.json`:
```json
"enum": [
  "reopen_loop",
  ...
  "approval_waiting_loop"
]
```

3. Update the weekly review schema and system prompt accordingly.

---

## 9. Customizing the Weekly Review Behavior

**Where:** `prompts/system-weekly-pattern-review.md` + `Prep Weekly Pattern Review` node

**To change how many top loop leaks are reported:**
Update the `maxItems` constraint in `weekly-pattern-review-output-schema.json`:
```json
"top_loop_leaks": {
  "maxItems": 5  // ← change from 3 to 5
}
```

**To change the confidence band thresholds:**
Edit the confidence band assignment rules in `system-weekly-pattern-review.md`:
```
- `high` if 5 or more loop results...  (default: 7)
- `medium` if 3–4 loop results...      (default: 3–6)
```

**To disable weekly review entirely:**
Remove the **If Weekly Review** node and connect **Human Review Override** directly to **Final Response**.

**To make weekly review automatic on a schedule:**
Add an **n8n Schedule Trigger** node set to `weekly` and connect it to a separate payload-building node that aggregates stored loop results.

---

## 10. Modifying Deterministic Logic Rules

**Where:** `Deterministic Friction Checks` node

The deterministic layer applies five rule-based overrides after AI interpretation. Each rule is documented with a comment. To disable a rule, comment it out:

```javascript
// Rule 1: Escalate severity (disabled)
// if (cluster.revisit_count >= 8 && ...) { ... }
```

To add a new rule:

```javascript
// Rule 6: Flag research_spiral if multiple distinct object names in events
const uniqueObjects = new Set(data.events.map(e => e.object_name));
if (uniqueObjects.size >= 5 && result.loop_type !== 'research_spiral') {
  deterministicFlags.push('LOOP_TYPE_CANDIDATE: 5+ distinct objects may indicate research_spiral');
}
```

Rules should be conservative. Do not add rules that override AI classification based on weak signal.

---

## 11. Modifying the Final Response Structure

**Where:** `Final Response` node

To add a field to the public response, add it to the `publicResponse` object in the Final Response node:

```javascript
publicResponse.analyzed_object = data.primary_cluster?.object_name || null;
publicResponse.event_count = data.event_count || 0;
```

To remove a field from the public response, delete it from the `publicResponse.loop_analysis` object.

All `_debug_` prefixed fields are automatically excluded because they are not explicitly assigned to `publicResponse`.

---

## 12. Adding a Notification for Human Review Cases

**Where:** After the `Human Review Override` node

When `needs_human_review` is true, the workflow currently passes through without notification. To add an alert:

1. Add a new **IF** node after **Human Review Override** with condition: `{{$json._debug_human_review_pending}} === true`
2. Connect the true branch to an **HTTP Request** node posting to your Slack webhook, email service, or notification system.
3. Connect the false branch back to **If Weekly Review**.

The payload for the notification should include:
- `user_id` from `$json.user_id`
- `review_reason` from `$json._debug_human_review_reason`
- `loop_type` from `$json._loop_analysis_result.loop_type`
- `severity_level` from `$json._loop_analysis_result.severity_level`
