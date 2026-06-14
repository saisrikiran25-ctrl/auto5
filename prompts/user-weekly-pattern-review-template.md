# User Prompt Template: Weekly Pattern Review

> This file is the user prompt template for the weekly pattern review call.
> Fields enclosed in `{{double_braces}}` are interpolated at runtime by the workflow.
> Do not send this template directly to the AI model. Use the populated version.

---

Analyze the following set of loop-analysis results from the past week and return a structured JSON weekly pattern review.

Do not hallucinate patterns not present in the data below.
Do not output prose, markdown, or explanation outside of the JSON object.
Return a single valid JSON object matching the expected output schema.

---

## Review Parameters

**USER:** {{user_id}}

**TIME WINDOW START:** {{time_window_start}}

ISO 8601 datetime string representing the start of the review period.

**TIME WINDOW END:** {{time_window_end}}

ISO 8601 datetime string representing the end of the review period.

**TOTAL LOOP RESULTS:** {{total_loop_results}}

Total number of single-loop analysis results included in this review.

**PREFERRED TONE:** {{preferred_tone}}

Accepted values: `direct` | `neutral` | `reflective`

Calibrate phrasing in narrative fields accordingly. Do not alter JSON structure based on tone.

---

## Loop Analysis Results

The following is the complete set of single-loop analysis results for this review period. Each entry is a JSON object produced by the single-loop analysis workflow stage.

```json
{{loop_analysis_results_json}}
```

Each result object contains:
- `loop_summary` — factual description of the pattern
- `loop_type` — classified loop type
- `friction_category` — classified friction category
- `severity_level` — none | low | medium | high | critical
- `estimated_attention_leak` — negligible | low | moderate | high | severe
- `supporting_patterns` — array of factual observations
- `likely_blocker` — identified blocker or null
- `recommended_intervention` — specific intervention
- `recommended_next_step` — smallest useful next move
- `confidence_band` — low | medium | high
- `needs_human_review` — boolean
- `review_reason` — string or null
- `object_name` — (where available) name of the repeated object
- `object_category` — (where available) category of the repeated object

---

## Synthesis Instructions

1. Review all loop results in the input. Do not add information not present in the results.

2. Identify friction categories that appear in two or more results. Report these as `repeated_friction_categories`.

3. Identify blocker descriptions that recur across two or more results. Report these as `recurring_blockers`. If none recur, return an empty array.

4. Identify the top 3 loop leaks by severity level for `top_loop_leaks`. If fewer than 3 results exist, include all of them.

5. Identify quick closure opportunities where a simple next action is likely to produce closure. These are typically loops with: a clear loop type, low revisit count, and no prior closure signal. Report these as `quick_closure_opportunities`. If none are apparent, return an empty array.

6. Identify the highest-severity patterns for `highest_severity_patterns`. Describe each in one factual sentence.

7. Recommend up to 5 specific interventions for the next period. Each intervention must be tied to a specific pattern or friction category identified in the results. Do not provide generic advice.

8. Assign the confidence band:
   - `high` if 7 or more loop results with predominantly medium or high confidence
   - `medium` if 3–6 loop results, or if results have mixed confidence
   - `low` if fewer than 3 results, or if most results had low confidence

9. Set `needs_human_review` to true if:
   - Multiple results flagged `needs_human_review: true`
   - A critical severity pattern is present with no clear intervention identified
   - The data is sparse or highly contradictory

10. Return a single valid JSON object only. No prose, no markdown, no code fences.

---

## Anti-Hallucination Constraints

- Every entry in `top_loop_leaks` must correspond to an actual result in the input.
- Every entry in `repeated_friction_categories` must appear in at least two input results.
- Every entry in `recurring_blockers` must be traceable to at least two input results.
- Every entry in `quick_closure_opportunities` must be grounded in a specific input result.
- Every entry in `next_week_interventions` must be tied to a specific pattern identified in the results.
- Do not invent loops not present in the input data.
- Do not extrapolate trends beyond the supplied time window.
- If the input contains only one result, scope the `review_summary` and all arrays accordingly and lower the confidence band.

---

## Expected Output Schema Reference

```json
{
  "review_summary": "string",
  "top_loop_leaks": [
    {
      "object_name": "string",
      "loop_type": "string",
      "severity_level": "string",
      "summary": "string"
    }
  ],
  "repeated_friction_categories": ["string"],
  "recurring_blockers": ["string"],
  "quick_closure_opportunities": ["string"],
  "highest_severity_patterns": ["string"],
  "next_week_interventions": ["string"],
  "confidence_band": "string",
  "needs_human_review": "boolean",
  "review_reason": "string or null"
}
```

Return JSON only.
