# User Prompt Template: Loop Analysis

> This file is the user prompt template for the single-loop analysis call.
> Fields enclosed in `{{double_braces}}` are interpolated at runtime by the workflow.
> Do not send this template directly to the AI model. Use the populated version.

---

Analyze the following repeated behavior cluster and return a structured JSON diagnostic.

Do not infer facts not supported by the event data below.
Do not output prose, markdown, or explanation outside of the JSON object.
Return a single valid JSON object matching the expected output schema.

---

## Analysis Parameters

**ANALYSIS MODE:** {{analysis_mode}}

Accepted values: `single_loop` | `weekly_review`

**TIME WINDOW START:** {{time_window_start}}

ISO 8601 datetime string. The earliest timestamp included in this analysis.

**TIME WINDOW END:** {{time_window_end}}

ISO 8601 datetime string. The latest timestamp included in this analysis.

**SOURCE SUMMARY:** {{source_summary}}

A brief description of the data sources included in this analysis (e.g., "browser tab activity, email draft events, task manager defer logs").

**EVENT COUNT:** {{event_count}}

Total number of events included in the normalized event set for this cluster.

**PREFERRED TONE:** {{preferred_tone}}

Accepted values: `direct` | `neutral` | `reflective`

Use this to calibrate phrasing in `recommended_intervention` and `recommended_next_step`. Direct = minimal hedging. Neutral = balanced. Reflective = slightly more exploratory framing. Do not alter JSON structure based on tone.

---

## Primary Cluster Data

**REPEATED OBJECT NAME:** {{repeated_object_name}}

The name or title of the digital object that was repeatedly accessed, edited, or deferred. Example: "Senior Product Manager - Acme Corp", "Q1 Report Draft v3", "Dentist Appointment Reminder".

**REPEATED OBJECT CATEGORY:** {{repeated_object_category}}

The category of the repeated object. Example: `job_listing`, `email_draft`, `document`, `task`, `note`, `calendar_item`, `tab`, `research_item`.

**REVISIT COUNT:** {{revisit_count}}

Total number of open/view/tab_open events for this object across all sessions.

**DRAFT EDIT COUNT:** {{draft_edit_count}}

Total number of edit/draft_edit/file_edit events for this object.

**REOPEN COUNT:** {{reopen_count}}

Total number of explicit reopen events (distinct from general open events).

**DEFER COUNT:** {{defer_count}}

Total number of defer/snooze/postpone events for this object.

**SEND COUNT:** {{send_count}}

Total number of send events for this object. Used to detect draft churn.

**SUBMIT COUNT:** {{submit_count}}

Total number of submit/apply events for this object. Used to detect revisit-without-commitment.

**DISCARD COUNT:** {{discard_count}}

Total number of discard/delete/archive events for this object.

**UNIQUE SESSIONS:** {{unique_sessions}}

Number of distinct session IDs in which this object was accessed.

**FIRST SEEN:** {{first_seen}}

Timestamp of the earliest event for this object in the time window.

**LAST SEEN:** {{last_seen}}

Timestamp of the most recent event for this object in the time window.

---

## Event Timeline

The chronological sequence of events for the primary cluster. Each line represents one event.

```
{{event_timeline}}
```

Format per line: `[timestamp] event_type (source, session: session_id)`

Example:
```
[2025-01-06T09:15:00Z] tab_open (browser, session: sess_001)
[2025-01-07T14:30:00Z] tab_open (browser, session: sess_002)
[2025-01-08T10:45:00Z] draft_edit (notes_app, session: sess_003)
[2025-01-08T11:05:00Z] tab_open (browser, session: sess_003)
[2025-01-09T16:00:00Z] tab_open (browser, session: sess_004)
```

---

## Analysis Instructions

1. Base your analysis strictly on the event data above. Do not infer facts, behaviors, or states not supported by the supplied signals.

2. Identify the loop type that best describes the observed pattern. Use exactly one of the valid loop_type values.

3. Identify the most likely friction category. Use exactly one of the valid friction_category values.

4. Estimate severity based on event frequency, cross-session recurrence, and lack of closure signals (send, submit, discard).

5. Estimate attention leak based on the number of sessions and events relative to any closure action.

6. List supporting patterns as short factual observations drawn directly from the event data.

7. Identify the likely blocker based on the combination of loop type, friction category, and event distribution. If the data does not support a specific inference, return null.

8. Recommend one specific intervention directly tied to the identified friction. Do not provide generic advice.

9. Recommend the single smallest next step the user could take to interrupt the loop.

10. Assign a confidence band based on event count, time window coverage, and pattern clarity.

11. Set needs_human_review to true if the pattern is ambiguous, contradictory, borderline, or if the inference required significant assumption. Set review_reason accordingly.

12. Do not output prose, markdown, headers, or code fences. Return a single valid JSON object only.

---

## Expected Output Schema Reference

```json
{
  "loop_summary": "string",
  "loop_type": "string",
  "friction_category": "string",
  "severity_level": "string",
  "estimated_attention_leak": "string",
  "supporting_patterns": ["string"],
  "likely_blocker": "string or null",
  "recommended_intervention": "string",
  "recommended_next_step": "string",
  "confidence_band": "string",
  "needs_human_review": "boolean",
  "review_reason": "string or null"
}
```

Return JSON only.
