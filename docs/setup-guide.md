# Setup Guide

Loop Leak Detector Automation — Complete Setup Instructions

---

## Prerequisites

Before beginning setup, ensure you have:

- A running n8n instance (self-hosted or n8n Cloud)
- n8n version 0.220.0 or later (for Code node v2 support)
- An active account with your preferred AI provider (OpenAI, Anthropic, Gemini, Mistral, or compatible)
- An API key from that provider (stored securely — never in this repo)
- Basic familiarity with n8n's interface for importing workflows and creating credentials

---

## Step 0: Configure Workflow Metadata (Before Import)

Before importing, open `workflow/loop-leak-detector.json` in a text editor and update these two fields:

### Update `meta.instanceId`

Locate the `meta` block near the bottom of the JSON:

```json
"meta": {
  "templateCredsSetupCompleted": false,
  "instanceId": "REPLACE_WITH_YOUR_INSTANCE_ID"
}
```

Replace `REPLACE_WITH_YOUR_INSTANCE_ID` with your actual n8n instance ID.

To find your instance ID in n8n:
1. Go to **Settings → About** in your n8n instance.
2. Copy the **Instance ID** value shown.
3. Paste it into the `instanceId` field in the workflow JSON.

If you leave this blank or use the placeholder, the workflow will still import and run correctly — but workflow templates with matching instance IDs integrate better with n8n Cloud and backup/restore features.

### Configure `settings.errorWorkflow` (Optional but Recommended)

Locate the `settings` block:

```json
"settings": {
  "executionOrder": "v1",
  "saveManualExecutions": true,
  "callerPolicy": "workflowsFromSameOwner",
  "errorWorkflow": "YOUR_ERROR_WORKFLOW_ID_OR_LEAVE_BLANK"
}
```

The `errorWorkflow` field is **intentionally left as a placeholder**. If it is empty or contains the placeholder string, n8n will silently swallow any unhandled execution errors — they will not produce notifications or alerts.

To enable error notifications:
1. Create a separate n8n workflow that handles error events (e.g., sends a Slack message or email).
2. Copy that workflow's ID from its URL (`/workflow/THE_ID_HERE`).
3. Paste the ID into `settings.errorWorkflow` before importing.

If you choose not to configure this, unhandled errors will be visible in the n8n execution history log but will not trigger any alert.

---

## Step 1: Import the Workflow

1. Open your n8n instance in a browser.
2. Navigate to **Workflows** in the left sidebar.
3. Click **+ Add Workflow** → **Import from File**.
4. Select `workflow/loop-leak-detector.json` from this repository (after completing Step 0).
5. Click **Import**.

The workflow will appear in your workflow list as **Loop Leak Detector Automation**.

> **Note:** After import, the workflow will be in **Inactive** state. Do not activate it until credentials are configured (Step 2–3).

---

## Step 2: Create AI Provider Credentials

Before the workflow can call your AI provider, you must create an HTTP Header Auth credential in n8n.

See [`docs/credential-setup.md`](credential-setup.md) for detailed credential setup instructions.

Quick summary:
1. Go to **Settings → Credentials → + Add Credential**
2. Select **HTTP Header Auth**
3. Set Header Name to `Authorization`
4. Set Header Value to `Bearer YOUR_API_KEY_HERE` (replace with your actual key)
5. Name the credential (e.g., `OpenAI API Key`)
6. Save

---

## Step 3: Configure the Provider Config Node

1. Open the imported workflow.
2. Click the **Provider Config** node.
3. In the JavaScript code, update the following values:

```javascript
const config = {
  // Your provider's chat completions endpoint
  apiUrl: 'https://api.openai.com/v1/chat/completions',

  // Your model identifier
  model: 'gpt-4o',

  // Must match the credential name you created in Step 2
  credentialName: 'OpenAI API Key',

  // Adjust as needed
  maxTokens: 1024,
  temperature: 0.2,

  // Clustering thresholds (adjust based on your data volume)
  thresholds: {
    minRevisitCount: 3,
    minDraftEdits: 3,
    minReopenCount: 2,
    minDeferCount: 2,
    lowSignalEventCount: 2
  }
};
```

4. Save the node.

---

## Step 4: Attach Credentials to AI Request Nodes

After configuring the Provider Config node, attach your credential to each AI HTTP request node:

1. Click the **AI: Analyze Loop** node.
2. In the **Authentication** section, select **Predefined Credential Type → HTTP Header Auth**.
3. Select the credential you created in Step 2 from the dropdown.
4. Save the node.

5. Repeat for the **AI: Weekly Pattern Review** node.

---

## Step 5: Activate the Workflow

1. Click the **Activate** toggle in the top-right corner of the workflow editor.
2. The webhook endpoint is now live.

To find your webhook URL:
1. Click the **Webhook** node.
2. Copy the **Test URL** (for testing) or **Production URL** (for live use).

The URL format will be:
```
https://your-n8n-instance.com/webhook/loop-leak-detector
```

---

## Step 6: Send a Sample Payload

Use the example payload from `schemas/activity-input-payload-example.json` to verify the workflow.

Using curl:
```bash
curl -X POST https://your-n8n-instance.com/webhook/loop-leak-detector \
  -H "Content-Type: application/json" \
  -d @schemas/activity-input-payload-example.json
```

Using Postman or Insomnia:
1. Create a new POST request.
2. Set the URL to your webhook endpoint.
3. Set the body to **Raw → JSON**.
4. Paste the contents of `schemas/activity-input-payload-example.json`.
5. Send.

---

## Step 7: Verify the Output

A successful response will return HTTP 200 with a JSON body matching the structure in `schemas/loop-analysis-output-schema.json`.

Example successful response shape:
```json
{
  "status": "success",
  "analysis_mode": "single_loop",
  "user_id": "user_abc123",
  "time_window_start": "2025-01-06T09:00:00Z",
  "time_window_end": "2025-01-12T23:59:59Z",
  "loop_analysis": {
    "loop_summary": "...",
    "loop_type": "revisit_without_commitment",
    "friction_category": "decision_avoidance",
    "severity_level": "medium",
    ...
  }
}
```

Verify that:
- No `_debug_` fields appear in the response
- `loop_type` is one of the valid enumerated values
- `severity_level` is one of: none, low, medium, high, critical
- `confidence_band` is one of: low, medium, high
- `needs_human_review` is a boolean

---

## Troubleshooting

### The webhook returns a 404

- Confirm the workflow is **Active** (toggle in top-right of editor).
- Confirm you are using the correct webhook URL (Test vs. Production).
- Check that the webhook path matches `loop-leak-detector`.

### The Normalize Events node throws a validation error

- Verify your input JSON contains all required top-level fields: `user_id`, `analysis_mode`, `time_window_start`, `time_window_end`, `events`.
- Verify each event object contains: `timestamp`, `source`, `event_type`, `object_name`.
- Validate your JSON with a linter before sending.

### The AI node returns a 401 or 403 error

- Confirm your API key is valid and has not expired.
- Confirm the credential in the HTTP Header Auth node is formatted as `Bearer YOUR_KEY`.
- Confirm the credential is attached to both AI request nodes.
- Check that the `apiUrl` in the Provider Config node matches your provider's actual endpoint.

### The Parse Loop Analysis node throws a JSON parse error

- Check the AI provider's raw response in the n8n execution log.
- The model may have returned markdown-wrapped JSON despite instructions. Update the system prompt or increase temperature slightly.
- The `needs_human_review` flag will be set to `true` and a fallback response will be returned automatically.
- For persistent issues, see the [customization guide](customization-guide.md) for prompt adjustment options.

### The output is classified as `low_signal_noise`

- Your event set may not meet the minimum thresholds configured in the Provider Config node.
- Expand the time window, add more event sources, or lower the `thresholds.lowSignalEventCount` value.
- Review which threshold rule is being triggered in the n8n execution log at the `Validate Cluster` node.

### The weekly review is not triggering

- Confirm `analysis_mode` is set to `"weekly_review"` in your input payload.
- The weekly review branch only activates when the If Weekly Review node condition is true.

---

### Errors occur but no notification arrives

The `errorWorkflow` field in the workflow JSON is intentionally left as a placeholder. An empty or placeholder value means n8n silently records the error in the execution history but sends no alert.

To enable error notifications after import:
1. In the n8n UI, open the Loop Leak Detector workflow.
2. Click the **Settings** (gear) icon in the top-right of the editor.
3. Under **Error Workflow**, select or paste your error-handler workflow ID.

Or edit `settings.errorWorkflow` in the workflow JSON before import (see Step 0).

---

## Extending the Workflow

- Add a **Slack** or **Email** notification node after the **Human Review Override** node to receive alerts when `needs_human_review` is true.
- Add a **Database** or **Airtable** node after the **Final Response** node to persist results.
- Add a **Schedule Trigger** node to run weekly reviews automatically on a schedule.
- See [`docs/customization-guide.md`](customization-guide.md) for full extension options.
