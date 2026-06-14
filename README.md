# Loop Leak Detector Automation

> **Behavioral-friction diagnostic automation for detecting repeated digital attention leaks with n8n + LLMs.**

A reusable n8n workflow template that identifies when you are stuck in a recurring digital pattern — reopening the same tab, re-editing the same draft, deferring the same task — and converts those patterns into structured, actionable diagnostics.

---

## What Is a "Loop Leak"?

A **loop leak** is a small recurring cycle of digital behavior that consumes attention without producing closure or forward movement.

| Pattern | What It Looks Like |
|---|---|
| `reopen_loop` | Same tab or file opened and closed repeatedly across sessions |
| `draft_churn_loop` | Same draft edited 4–6 times with no send or discard |
| `revisit_without_commitment` | Same job listing, option set, or reference revisited across days with no decision |
| `defer_reopen_cycle` | Task deferred repeatedly, then reopened — never resolved |
| `research_spiral` | Multiple related items opened across sessions with no convergence |
| `context_switch_drag` | Same object interrupted and re-entered repeatedly without completion |
| `unresolved_open_loop` | Object stays persistently open across a long window with zero action |
| `low_signal_noise` | Pattern is too weak to interpret meaningfully |

These are not failures. They are **friction signals** — indicators that something is blocking forward movement. This automation helps you see the pattern, identify the likely blocker, and find the smallest useful next move.

---

## What This Is — and Is Not

| This IS | This is NOT |
|---|---|
| A behavioral-friction diagnostic | A task manager |
| A loop-detection workflow | A journaling tool |
| A personal operations diagnostic tool | A therapy assistant |
| A reusable automation template | A browser extension UI |
| An attention-leak analysis system | A generic productivity chatbot |

---

## Who This Is For

- Knowledge workers who want visibility into their own behavioral friction
- Personal operations practitioners using n8n for self-tracking and diagnostics
- Developers building custom digital behavior analytics pipelines
- Automation builders who want a production-quality repeated-behavior detection template
- Anyone who wants to understand where their attention is leaking before it becomes a habit

---

## Key Features

- **Repeated behavior clustering** — Groups events by object, category, and session across a configurable time window
- **Loop type classification** — Classifies patterns into 8 named loop types with strict enumeration
- **AI-powered loop interpretation** — Uses your LLM provider to analyze clusters and identify friction, severity, and recommended intervention
- **First-class noise suppression** — Low-signal clusters are filtered before reaching the AI layer; no over-reading of normal browsing
- **Closure event modeling** — Tracks send, submit, archive, and discard events to detect when a loop has genuinely ended
- **Severity calibration by object type** — `job_listing` + repeated revisit + no apply is weighted differently from casual note browsing
- **Deterministic friction checks** — Rule-based validation layer runs after AI interpretation to prevent hallucinated overrides
- **Human review routing** — Flags ambiguous or critical-severity outputs for optional human override
- **Weekly pattern review** — Optional second-pass synthesis across multiple loop results
- **Clean output contract** — Final response strips all internal fields; all internal-only diagnostic fields use the `_debug_` prefix and are removed from public output
- **Provider-agnostic** — Any HTTP-compatible LLM provider (OpenAI, Anthropic, Gemini, Mistral, local models)
- **No hardcoded secrets** — You bring your own credentials

---

## Workflow Overview

```
Webhook
  └─▶ Provider Config
        └─▶ Normalize Events
              └─▶ Cluster Repeated Behavior
                    └─▶ Closure Event Detection
                          └─▶ Severity Calibration (by object type)
                                └─▶ Validate Cluster
                                      ├─▶ [Low Signal] → Low Signal Response
                                      └─▶ [Valid] → Prep Loop Analysis Request
                                                        └─▶ AI: Analyze Loop
                                                              └─▶ Parse Loop Analysis
                                                                    └─▶ Deterministic Friction Checks
                                                                          └─▶ Human Review Override
                                                                                ├─▶ [weekly_review] → Prep Weekly Pattern Review
                                                                                │                         └─▶ AI: Weekly Pattern Review
                                                                                │                               └─▶ Parse Weekly Pattern Review
                                                                                └─▶ Final Response
                                                                                      └─▶ Respond to Webhook
```

Each stage is documented in [`assets/architecture-overview.md`](assets/architecture-overview.md).

---

## Directory Structure

```
loop-leak-detector-automation/
├── README.md                                    ← This file
├── workflow/
│   └── loop-leak-detector.json                  ← Importable n8n workflow
├── prompts/
│   ├── system-loop-analysis.md                  ← System prompt: single loop analysis
│   ├── user-loop-analysis-template.md           ← User prompt template: single loop
│   ├── system-weekly-pattern-review.md          ← System prompt: weekly review
│   └── user-weekly-pattern-review-template.md  ← User prompt template: weekly review
├── schemas/
│   ├── activity-input-payload-example.json      ← Sample input payload
│   ├── loop-analysis-output-schema.json         ← Output contract: single loop
│   └── weekly-pattern-review-output-schema.json ← Output contract: weekly review
├── docs/
│   ├── setup-guide.md                           ← Full setup instructions
│   ├── credential-setup.md                      ← Credential configuration guide
│   ├── customization-guide.md                   ← How to customize the workflow
│   └── testing-guide.md                         ← Test cases and expected behavior
└── assets/
    └── architecture-overview.md                 ← Workflow architecture documentation
```

---

## Setup Summary

1. Import `workflow/loop-leak-detector.json` into your n8n instance
2. Create your AI provider credentials in n8n (see [`docs/credential-setup.md`](docs/credential-setup.md))
3. Configure the `Provider Config` node with your API endpoint and model
4. Send a sample payload to the webhook endpoint
5. Verify the output matches the schema in [`schemas/loop-analysis-output-schema.json`](schemas/loop-analysis-output-schema.json)

Full setup instructions: [`docs/setup-guide.md`](docs/setup-guide.md)

---

## Credential Note

**This repository does not contain any API keys, secrets, or credentials.**

You must supply your own AI provider credentials. The workflow accepts any HTTP-compatible LLM API. Credentials are configured inside n8n and are never stored in the workflow JSON.

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
    },
    {
      "timestamp": "2025-01-09T08:00:00Z",
      "source": "task_manager",
      "event_type": "defer",
      "object_name": "Senior Product Manager - Acme Corp",
      "object_category": "job_listing",
      "session_id": "sess_003"
    }
  ]
}
```

Full example: [`schemas/activity-input-payload-example.json`](schemas/activity-input-payload-example.json)

---

## Example Output

```json
{
  "status": "success",
  "analysis_mode": "single_loop",
  "user_id": "user_abc123",
  "time_window_start": "2025-01-06T09:00:00Z",
  "time_window_end": "2025-01-12T23:59:59Z",
  "loop_analysis": {
    "loop_summary": "The object 'Senior Product Manager - Acme Corp' was opened 5 times across 5 sessions in a 7-day window with no application or discard action recorded.",
    "loop_type": "revisit_without_commitment",
    "friction_category": "decision_avoidance",
    "severity_level": "medium",
    "estimated_attention_leak": "moderate",
    "supporting_patterns": [
      "5 opens across 5 separate sessions",
      "0 application events",
      "0 discard events",
      "1 defer event on day 4 with resumed revisit",
      "Consistent cross-session recurrence over 7 days"
    ],
    "likely_blocker": "Unclear application criteria or unresolved uncertainty about fit",
    "recommended_intervention": "Define a binary apply/discard rule based on one specific criterion before next open",
    "recommended_next_step": "Open the listing one final time with a preset 5-minute decision timer",
    "confidence_band": "medium",
    "needs_human_review": false,
    "review_reason": null
  }
}
```

Full schema: [`schemas/loop-analysis-output-schema.json`](schemas/loop-analysis-output-schema.json)

---

## Intervention Style Examples

The automation returns specific, friction-matched interventions — not generic advice. Here are representative examples across loop types:

| Loop Type | Intervention Style | Example |
|---|---|---|
| `revisit_without_commitment` | Send/archive threshold | "Define a binary apply-or-discard rule before the next open" |
| `draft_churn_loop` | One-final-review rule | "Set a word-count cap or send-by date — no further edits after that date" |
| `defer_reopen_cycle` | Forcing function | "Require a named dependency before the next defer is allowed" |
| `reopen_loop` | Yes/no decision checkpoint | "Convert the next reopen into a 2-minute triage: close permanently or schedule a time block" |
| `low_signal_noise` | Noise suppression | "Archive as low-signal — not a meaningful attention leak at this volume" |
| `research_spiral` | Convergence gate | "Create one clarifying question that, when answered, ends the research phase" |

---

## Customization Summary

You can customize:
- LLM provider, model, and API endpoint
- Event clustering thresholds (revisit count, draft edits, reopen count)
- Severity calibration rules by `object_category`
- Closure event vocabulary (send, submit, archive, discard)
- Friction label vocabulary
- Intervention wording style
- Output tone (`direct`, `neutral`, `reflective`)
- Weekly review behavior
- Deterministic logic thresholds

Full customization guide: [`docs/customization-guide.md`](docs/customization-guide.md)

---

## Testing Summary

Test cases in [`docs/testing-guide.md`](docs/testing-guide.md) include:

| Test | Scenario |
|---|---|
| Test 1 | Happy-path repeated revisit loop |
| Test 2 | Draft churn / no-send loop |
| Test 3 | Low-signal noisy activity — AI not called |
| Test 4 | Repeated defer/reopen loop |
| Test 5 | Urgent object with no action |
| Test 6 | Malformed model output — graceful degradation |
| Test 7 | Weekly review mode |

---

## Limitations

This automation works best under specific conditions. Be aware of the following constraints:

- **Requires structured event logs** — The workflow depends on well-formed event data with consistent `object_name`, `object_category`, and `event_type` fields. Unstructured or inconsistent source data will degrade cluster quality.
- **Weak on sparse or noisy data** — Fewer than 3–4 events per object cluster will trigger the low-signal gate and return no AI analysis. This is by design, not a bug.
- **Quality depends on event normalization** — If the same object appears under slightly different names across events (e.g., `"Q1 Report"` vs `"Q1 Report Draft"`), the clustering layer will treat them as separate objects. Normalize names at the source before sending.
- **Not a clinical or psychological tool** — Loop type and friction category labels are behavioral descriptors, not diagnostic categories. The system is intentionally constrained from psychological inference.
- **Not reliable below minimum event density** — The automation is designed for knowledge workers with moderate-to-high digital activity. Very low-frequency activity will produce mostly low-signal results.
- **LLM quality is provider-dependent** — Output quality varies with model. Smaller or lower-capability models may produce less precise friction classification. The deterministic layer partially compensates for this, but does not eliminate provider-quality variance.
- **No memory across sessions by default** — Each workflow invocation is stateless. The workflow does not track loop history across multiple separate calls. Weekly review is only as good as the data supplied in the single payload.

---

## Disclaimer

This automation is a diagnostic tool, not a decision-making authority. Its outputs are structured interpretations of behavioral signals — they do not replace human judgment, professional advice, or personal context. The workflow is intentionally cautious about psychological inference. It will not diagnose conditions, make claims about mental state, or issue prescriptive guidance beyond the scope of the data supplied.

Use it as a signal, not a verdict.

---

## License

MIT License. Use freely. Modify as needed. Do not redistribute with embedded credentials.

---

*Loop Leak Detector Automation — behavioral-friction diagnostics for knowledge workers.*
