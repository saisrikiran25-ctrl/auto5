# Testing Guide

Loop Leak Detector Automation — Test Cases and Expected Behavior

---

## Overview

This guide provides structured test cases for verifying the Loop Leak Detector Automation. Each test case includes:
- A description of the scenario
- The input payload or modification to the base payload
- The expected behavior at each stage
- The expected output fields and values
- Notes on edge cases or failure modes

Use these tests after initial setup and whenever you modify the workflow, prompts, or thresholds.

---

## Base Testing Setup

Before running test cases, confirm:
1. The workflow is active and the webhook URL is accessible.
2. A valid AI provider credential is configured.
3. The Provider Config node uses default threshold values:
   - `minRevisitCount: 3`
   - `minDraftEdits: 3`
   - `minReopenCount: 2`
   - `minDeferCount: 2`
   - `lowSignalEventCount: 2`

All test payloads should be sent as `POST` requests to your webhook URL with `Content-Type: application/json`.

---

## Test 1: Happy-Path Repeated Revisit Loop

### Scenario

A user opens the same job listing 5 times across 5 separate sessions in a 7-day window. They defer it once. They never apply or discard.

### Input Payload

```json
{
  "user_id": "test_user_001",
  "analysis_mode": "single_loop",
  "time_window_start": "2025-01-06T09:00:00Z",
  "time_window_end": "2025-01-12T23:59:59Z",
  "preferred_tone": "direct",
  "events": [
    {"timestamp": "2025-01-06T09:15:00Z", "source": "browser", "event_type": "tab_open", "object_name": "Senior PM - Acme Corp", "object_category": "job_listing", "session_id": "sess_001"},
    {"timestamp": "2025-01-07T14:30:00Z", "source": "browser", "event_type": "tab_open", "object_name": "Senior PM - Acme Corp", "object_category": "job_listing", "session_id": "sess_002"},
    {"timestamp": "2025-01-08T10:45:00Z", "source": "browser", "event_type": "tab_open", "object_name": "Senior PM - Acme Corp", "object_category": "job_listing", "session_id": "sess_003"},
    {"timestamp": "2025-01-09T08:00:00Z", "source": "task_manager", "event_type": "defer", "object_name": "Senior PM - Acme Corp", "object_category": "job_listing", "session_id": "sess_004"},
    {"timestamp": "2025-01-10T11:20:00Z", "source": "browser", "event_type": "tab_open", "object_name": "Senior PM - Acme Corp", "object_category": "job_listing", "session_id": "sess_005"},
    {"timestamp": "2025-01-11T16:00:00Z", "source": "browser", "event_type": "tab_open", "object_name": "Senior PM - Acme Corp", "object_category": "job_listing", "session_id": "sess_006"}
  ]
}
```

### Expected Behavior

| Stage | Expected |
|---|---|
| Normalize Events | Passes. 6 events normalized. |
| Cluster Repeated Behavior | One cluster: `Senior PM - Acme Corp` with revisit_count=5, defer_count=1, submit_count=0 |
| Validate Cluster | `cluster_is_valid: true` (revisit_count meets minRevisitCount) |
| If Valid | Routes to AI analysis branch |
| AI: Analyze Loop | Returns JSON with `loop_type: revisit_without_commitment` |
| Deterministic Checks | No overrides triggered (pattern is clear) |
| Final Response | Clean response, no `_debug_` fields |

### Expected Output Fields

```json
{
  "loop_type": "revisit_without_commitment",
  "friction_category": "decision_avoidance",
  "severity_level": "low or medium",
  "estimated_attention_leak": "low or moderate",
  "needs_human_review": false
}
```

### Pass Criteria

- `loop_type` is `revisit_without_commitment`
- `submit_count` evidence cited in `supporting_patterns`
- `recommended_intervention` references the need for a decision rule
- No `_debug_` fields in response

---

## Test 2: Draft Churn / No-Send Loop

### Scenario

A user edits the same email draft 5 times across 4 sessions with 0 send events.

### Input Payload

```json
{
  "user_id": "test_user_002",
  "analysis_mode": "single_loop",
  "time_window_start": "2025-01-06T09:00:00Z",
  "time_window_end": "2025-01-10T23:59:59Z",
  "preferred_tone": "direct",
  "events": [
    {"timestamp": "2025-01-06T10:00:00Z", "source": "email_client", "event_type": "draft_edit", "object_name": "Partnership Proposal - DataCo", "object_category": "email_draft", "session_id": "sess_001"},
    {"timestamp": "2025-01-07T14:00:00Z", "source": "email_client", "event_type": "draft_edit", "object_name": "Partnership Proposal - DataCo", "object_category": "email_draft", "session_id": "sess_002"},
    {"timestamp": "2025-01-08T09:30:00Z", "source": "email_client", "event_type": "draft_edit", "object_name": "Partnership Proposal - DataCo", "object_category": "email_draft", "session_id": "sess_003"},
    {"timestamp": "2025-01-09T16:15:00Z", "source": "email_client", "event_type": "draft_edit", "object_name": "Partnership Proposal - DataCo", "object_category": "email_draft", "session_id": "sess_004"},
    {"timestamp": "2025-01-10T11:00:00Z", "source": "email_client", "event_type": "draft_edit", "object_name": "Partnership Proposal - DataCo", "object_category": "email_draft", "session_id": "sess_004"}
  ]
}
```

### Expected Behavior

| Stage | Expected |
|---|---|
| Cluster Repeated Behavior | draft_edit_count=5, send_count=0 |
| Validate Cluster | `cluster_is_valid: true` (draft_edit_count meets minDraftEdits) |
| Deterministic Friction Checks | Rule 2 triggers: `loop_type` overridden to `draft_churn_loop` if AI did not already classify it |
| Final Response | `loop_type: draft_churn_loop` |

### Expected Output Fields

```json
{
  "loop_type": "draft_churn_loop",
  "friction_category": "perfectionism_stall",
  "severity_level": "low or medium",
  "recommended_intervention": "references send threshold or deadline"
}
```

### Pass Criteria

- `loop_type` is `draft_churn_loop` (either from AI or deterministic override)
- `send_count: 0` evidence in `supporting_patterns`
- `recommended_intervention` does not say "make a to-do list" or similar generic advice

---

## Test 3: Low-Signal Noisy Activity Test

### Scenario

A user sends only 2 events with no repeated object. Signal is below threshold.

### Input Payload

```json
{
  "user_id": "test_user_003",
  "analysis_mode": "single_loop",
  "time_window_start": "2025-01-06T09:00:00Z",
  "time_window_end": "2025-01-07T23:59:59Z",
  "preferred_tone": "direct",
  "events": [
    {"timestamp": "2025-01-06T10:00:00Z", "source": "browser", "event_type": "tab_open", "object_name": "Random Article - Tech Blog", "object_category": "article", "session_id": "sess_001"},
    {"timestamp": "2025-01-07T09:00:00Z", "source": "browser", "event_type": "tab_open", "object_name": "Different Article - Dev Blog", "object_category": "article", "session_id": "sess_002"}
  ]
}
```

### Expected Behavior

| Stage | Expected |
|---|---|
| Cluster Repeated Behavior | Two clusters, each with 1 event |
| Validate Cluster | `cluster_is_valid: false` (total_events <= lowSignalEventCount) |
| If Valid | Routes to Low Signal Response branch |
| AI nodes | Not called |
| Final Response | Returns low-signal structured response |

### Expected Output Fields

```json
{
  "loop_type": "low_signal_noise",
  "friction_category": "insufficient_data",
  "severity_level": "none",
  "estimated_attention_leak": "negligible",
  "needs_human_review": false
}
```

### Pass Criteria

- AI nodes are NOT called (no API cost incurred)
- Response is returned immediately with `loop_type: low_signal_noise`
- `recommended_intervention` suggests expanding the time window or adding more event sources

---

## Test 4: Repeated Defer/Reopen Loop

### Scenario

A task is deferred 3 times and reopened 3 times across 5 sessions with no completion event.

### Input Payload

```json
{
  "user_id": "test_user_004",
  "analysis_mode": "single_loop",
  "time_window_start": "2025-01-06T09:00:00Z",
  "time_window_end": "2025-01-12T23:59:59Z",
  "preferred_tone": "direct",
  "events": [
    {"timestamp": "2025-01-06T09:00:00Z", "source": "task_manager", "event_type": "item_open", "object_name": "File Q4 VAT Return", "object_category": "task", "session_id": "sess_001"},
    {"timestamp": "2025-01-06T09:05:00Z", "source": "task_manager", "event_type": "defer", "object_name": "File Q4 VAT Return", "object_category": "task", "session_id": "sess_001"},
    {"timestamp": "2025-01-07T10:00:00Z", "source": "task_manager", "event_type": "reopen", "object_name": "File Q4 VAT Return", "object_category": "task", "session_id": "sess_002"},
    {"timestamp": "2025-01-07T10:10:00Z", "source": "task_manager", "event_type": "defer", "object_name": "File Q4 VAT Return", "object_category": "task", "session_id": "sess_002"},
    {"timestamp": "2025-01-09T11:00:00Z", "source": "task_manager", "event_type": "reopen", "object_name": "File Q4 VAT Return", "object_category": "task", "session_id": "sess_003"},
    {"timestamp": "2025-01-09T11:15:00Z", "source": "task_manager", "event_type": "defer", "object_name": "File Q4 VAT Return", "object_category": "task", "session_id": "sess_003"},
    {"timestamp": "2025-01-11T08:30:00Z", "source": "task_manager", "event_type": "reopen", "object_name": "File Q4 VAT Return", "object_category": "task", "session_id": "sess_004"},
    {"timestamp": "2025-01-12T14:00:00Z", "source": "task_manager", "event_type": "item_open", "object_name": "File Q4 VAT Return", "object_category": "task", "session_id": "sess_005"}
  ]
}
```

### Expected Behavior

| Stage | Expected |
|---|---|
| Cluster Repeated Behavior | defer_count=3, reopen_count=3, revisit_count=2 |
| Validate Cluster | `cluster_is_valid: true` |
| Deterministic Checks | Rule 3 candidate flag for `defer_reopen_cycle` |
| Final Response | `loop_type: defer_reopen_cycle` |

### Expected Output Fields

```json
{
  "loop_type": "defer_reopen_cycle",
  "friction_category": "decision_avoidance or dependency_block",
  "severity_level": "medium",
  "recommended_intervention": "references a specific forcing function or dependency resolution"
}
```

### Pass Criteria

- `loop_type` is `defer_reopen_cycle`
- Supporting patterns cite the defer and reopen event counts
- `recommended_next_step` is specific and actionable

---

## Test 5: Urgent Object With No Action

### Scenario

An object with "urgent" or deadline-adjacent terms in the name has 4 revisits, 0 action events, and revisits span 5 days.

### Input Payload

Modify Test 1's payload:
- Change `object_name` to `"URGENT: Q1 Board Report - Final Draft"`
- Add 4 tab_open events across 5 sessions
- 0 edit, send, or submit events

### Expected Behavior

- AI should identify high severity given the object name signal and lack of closure
- Deterministic Rule 1 should escalate severity if revisit_count >= 8
- `needs_human_review` may be set if the pattern is ambiguous

### Pass Criteria

- `severity_level` is `medium` or `high`
- `recommended_intervention` is specific (references the object's apparent deadline nature)
- No generic advice in the output

---

## Test 6: Malformed Model Output Test

### Scenario

Simulate a parse failure by temporarily pointing the workflow at an endpoint that returns non-JSON text, or by modifying the Parse Loop Analysis node to force a JSON.parse error on the test content.

> **Note:** This test verifies graceful degradation, not AI quality. Do not test this against your real provider.

### How to Trigger

In the **Parse Loop Analysis** node, temporarily add this at the top of the try block:
```javascript
throw new Error('Simulated parse failure for testing');
```

Send any valid payload.

### Expected Behavior

| Stage | Expected |
|---|---|
| Parse Loop Analysis | Catches the error, constructs fallback response |
| Deterministic Checks | Runs on fallback response |
| Final Response | Returns structured response with `needs_human_review: true` |

### Expected Output Fields

```json
{
  "loop_type": "unresolved_open_loop",
  "confidence_band": "low",
  "needs_human_review": true,
  "review_reason": "JSON parse failure: ..."
}
```

### Pass Criteria

- Workflow does not crash or return HTTP 500
- Response is a valid JSON object
- `needs_human_review: true` with a non-null `review_reason`
- Remove the simulated error after testing

---

## Test 7: Weekly Review Mode

### Scenario

Submit a payload with `analysis_mode: weekly_review` and enough events to produce a valid cluster.

### Input Payload

Use the same payload as Test 1 but change:
```json
"analysis_mode": "weekly_review"
```

### Expected Behavior

| Stage | Expected |
|---|---|
| If Weekly Review | Routes to Prep Weekly Pattern Review |
| AI: Weekly Pattern Review | Called with weekly synthesis prompt |
| Parse Weekly Pattern Review | Returns weekly review JSON |
| Final Response | Includes both `loop_analysis` and `weekly_review` fields |

### Pass Criteria

- Response contains both `loop_analysis` and `weekly_review` keys
- `weekly_review.top_loop_leaks` is an array
- `weekly_review.next_week_interventions` contains at least one item
- No `_debug_` fields in response
