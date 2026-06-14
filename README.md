# Loop Leak Detector Automation

> A behavioral-friction diagnostic automation that identifies repeated digital patterns, classifies them as attention leaks, and surfaces the smallest useful intervention.

---

## What This Is

**Loop Leak Detector** is a reusable n8n automation template that detects when you are stuck in a recurring digital pattern without producing closure.

It is not a task manager. It is not a journaling tool. It is not a chatbot.

It is a structured behavioral-friction diagnostic — designed to identify the invisible cycles of repeated, unresolved digital action that quietly drain your attention.

---

## What Is a "Loop Leak"?

A **loop leak** is a small recurring cycle of digital behavior that consumes attention without producing closure or forward movement.

Examples:
- Reopening the same browser tab across multiple sessions without acting on it
- Editing the same draft email or document repeatedly without sending or discarding
- Revisiting the same job post, listing, or reference item many times without deciding
- Deferring the same task or item across multiple defer cycles
- Bouncing between the same set of notes, options, or references without resolution
- Re-reading the same option set without committing to a choice
- Reopening and re-closing the same object across separate work sessions

These behaviors are not failures. They are friction signals. They indicate something is blocking forward movement — often uncertainty, unclear next steps, perfectionism, avoidance, or an unresolved dependency.

This automation helps you see the pattern, understand the likely blocker, and identify the smallest useful next move.

---

## Who This Is For

- Knowledge workers who want visibility into their own behavioral friction
- Personal operations practitioners who use n8n for self-tracking and diagnostics
- Developers building custom digital behavior analytics pipelines
- Automation builders who want a production-quality workflow template for repeated-behavior detection
- Anyone who wants to understand where their attention is leaking before it becomes a habit

---

## Key Features

- **Repeated behavior clustering** — Groups related events by object, category, and session across a configurable time window
- **Loop type classification** — Classifies patterns as `reopen_loop`, `draft_churn_loop`, `revisit_without_commitment`, `defer_reopen_cycle`, `research_spiral`, `context_switch_drag`, `low_signal_noise`, or `unresolved_open_loop`
- **AI-powered loop interpretation** — Uses your preferred LLM provider to analyze the cluster and identify friction, severity, and recommended intervention
- **Deterministic friction checks** — Rule-based validation layer runs before and after AI interpretation to prevent over-inference
- **Human review routing** — Flags ambiguous or high-severity outputs for optional human override
- **Weekly pattern review** — Optional second-pass analysis that synthesizes multiple loop results into a structured weekly diagnostic
- **Clean output contract** — Final response strips all internal and debug fields before returning to the caller
- **Provider-agnostic** — Works with any HTTP-compatible LLM provider (OpenAI, Anthropic, Gemini, Mistral, local models, etc.)
- **No hardcoded secrets** — You bring your own credentials

---

## Workflow Overview

```
Webhook → Provider Config → Normalize Events → Cluster Repeated Behavior
→ Validate Cluster → [If Valid] → Prep Loop Analysis Request → AI: Analyze Loop
→ Parse Loop Analysis → Deterministic Friction Checks → Human Review Override
→ [Optional] Prep Weekly Pattern Review → AI: Weekly Pattern Review
→ Parse Weekly Pattern Review → Final Response → Respond to Webhook
```

Each stage is documented in [`assets/architecture-overview.md`](assets/architecture-overview.md).

---

## Directory Structure

```
loop-leak-detector-automation/
├── README.md                                  ← This file
├── workflow/
│   └── loop-leak-detector.json                ← Importable n8n workflow
├── prompts/
│   ├── system-loop-analysis.md                ← System prompt: single loop analysis
│   ├── user-loop-analysis-template.md         ← User prompt template: single loop
│   ├── system-weekly-pattern-review.md        ← System prompt: weekly review
│   └── user-weekly-pattern-review-template.md ← User prompt template: weekly review
├── schemas/
│   ├── activity-input-payload-example.json    ← Sample input payload
│   ├── loop-analysis-output-schema.json       ← Output contract: single loop
│   └── weekly-pattern-review-output-schema.json ← Output contract: weekly review
├── docs/
│   ├── setup-guide.md                         ← Full setup instructions
│   ├── credential-setup.md                    ← Credential configuration guide
│   ├── customization-guide.md                 ← How to customize the workflow
│   └── testing-guide.md                       ← Test cases and expected behavior
└── assets/
    └── architecture-overview.md               ← Workflow architecture documentation
```

---

## Setup Summary

1. Import `workflow/loop-leak-detector.json` into your n8n instance
2. Create your AI provider credentials in n8n (see [`docs/credential-setup.md`](docs/credential-setup.md))
3. Configure the `Provider Config` node with your API endpoint and model
4. Send a sample payload to the webhook endpoint using the example in [`schemas/activity-input-payload-example.json`](schemas/activity-input-payload-example.json)
5. Verify the output matches the schema in [`schemas/loop-analysis-output-schema.json`](schemas/loop-analysis-output-schema.json)

Full setup instructions: [`docs/setup-guide.md`](docs/setup-guide.md)

---

## Credential Note

**This repository does not contain any API keys, secrets, or credentials.**

You must supply your own AI provider credentials. The workflow is designed to accept any HTTP-compatible LLM API. Credentials are configured inside n8n and are never stored in the workflow JSON.

See [`docs/credential-setup.md`](docs/credential-setup.md) for step-by-step instructions.

---

## Example Input

```json
{
  "user_id": "user_abc123",
  "analysis_mode": "single_loop",
  "time_window_start": "2025-01-06T09:00:00Z",
  "time_window_end": "2025-01-12T23:59:59Z",
  "preferred_tone": "direct",
  "events": [
    {
      "timestamp": "2025-01-06T09:15:00Z",
      "source": "browser",
      "event_type": "tab_open",
      "object_name": "Senior Product Manager - Acme Corp",
      "object_category": "job_listing",
      "session_id": "sess_001"
    },
    {
      "timestamp": "2025-01-07T14:30:00Z",
      "source": "browser",
      "event_type": "tab_open",
      "object_name": "Senior Product Manager - Acme Corp",
      "object_category": "job_listing",
      "session_id": "sess_002"
    }
  ]
}
```

Full example: [`schemas/activity-input-payload-example.json`](schemas/activity-input-payload-example.json)

---

## Example Output

```json
{
  "loop_summary": "The object 'Senior Product Manager - Acme Corp' was opened 5 times across 5 sessions in a 7-day window with no application or discard action recorded.",
  "loop_type": "revisit_without_commitment",
  "friction_category": "decision_avoidance",
  "severity_level": "medium",
  "estimated_attention_leak": "moderate",
  "supporting_patterns": [
    "5 opens across 5 separate sessions",
    "0 application events",
    "0 discard events",
    "consistent cross-session recurrence"
  ],
  "likely_blocker": "Unclear application criteria or unresolved uncertainty about fit",
  "recommended_intervention": "Define a binary apply/discard rule based on one specific criterion before next open",
  "recommended_next_step": "Open the listing one final time with a preset 5-minute decision timer",
  "confidence_band": "medium",
  "needs_human_review": false,
  "review_reason": null
}
```

Full schema: [`schemas/loop-analysis-output-schema.json`](schemas/loop-analysis-output-schema.json)

---

## Customization Summary

You can customize:
- LLM provider, model, and API endpoint
- Event clustering thresholds (revisit count, draft edits, reopen count)
- Friction label vocabulary
- Intervention wording style
- Output tone (direct, neutral, reflective)
- Weekly review behavior
- Deterministic logic rules

Full customization guide: [`docs/customization-guide.md`](docs/customization-guide.md)

---

## Testing Summary

Test cases provided in [`docs/testing-guide.md`](docs/testing-guide.md) include:
- Happy-path repeated revisit loop
- Draft churn / no-send loop
- Low-signal noisy activity
- Repeated defer/reopen loop
- Urgent object with no action
- Malformed model output handling

---

## Disclaimer

This automation is a diagnostic tool, not a decision-making authority. Its outputs are structured interpretations of behavioral signals — they do not replace human judgment, professional advice, or personal context. The workflow is intentionally cautious about psychological inference. It will not diagnose conditions, make claims about mental state, or issue prescriptive guidance beyond the scope of the data supplied.

Use it as a signal, not a verdict.

---

## License

MIT License. Use freely. Modify as needed. Do not redistribute with embedded credentials.

---

*Loop Leak Detector Automation — behavioral-friction diagnostics for knowledge workers.*
