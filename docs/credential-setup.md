# Credential Setup Guide

Loop Leak Detector Automation — AI Provider Credential Configuration

---

## Important: This Repository Does Not Include API Keys

**No API key, secret, token, or credential of any kind is included in this repository.**

You must supply your own AI provider credentials. This is by design. The workflow is structured so that credentials are stored entirely within n8n's credential vault and are never written to any workflow JSON, prompt file, schema file, or documentation file.

If you have found what appears to be a real API key anywhere in this repository, it is an error. Do not use it. Rotate it immediately if it belongs to you.

---

## Credential Architecture

The Loop Leak Detector Automation uses **n8n's HTTP Header Auth credential type** to authenticate against your AI provider's API.

This approach works with any HTTP-compatible LLM provider, including:

| Provider | API Endpoint Example |
|---|---|
| OpenAI | `https://api.openai.com/v1/chat/completions` |
| Anthropic | `https://api.anthropic.com/v1/messages` |
| Google Gemini | `https://generativelanguage.googleapis.com/v1beta/openai/chat/completions` |
| Mistral | `https://api.mistral.ai/v1/chat/completions` |
| Groq | `https://api.groq.com/openai/v1/chat/completions` |
| Together AI | `https://api.together.xyz/v1/chat/completions` |
| Local (Ollama) | `http://localhost:11434/v1/chat/completions` |
| Any OpenAI-compatible endpoint | Varies |

---

## Step-by-Step: Creating Your Credential in n8n

### 1. Open the Credentials Panel

1. In your n8n instance, click the **Settings** gear icon in the bottom-left sidebar.
2. Select **Credentials**.
3. Click **+ Add Credential** in the top-right corner.

### 2. Select the Credential Type

1. In the credential type search box, type `HTTP Header Auth`.
2. Select **HTTP Header Auth** from the results.
3. Click **Continue**.

### 3. Configure the Credential

Fill in the following fields:

**Credential Name:**
Choose a name that identifies the provider and purpose. Examples:
- `OpenAI - Loop Leak Detector`
- `Anthropic API Key`
- `Mistral Production Key`

This name is what you will reference in the Provider Config node's `credentialName` field.

**Name (Header Name):**
Enter: `Authorization`

This is the HTTP request header that will be sent with every AI request.

**Value (Header Value):**
Enter your API key in the following format:

```
Bearer YOUR_API_KEY_HERE
```

Replace `YOUR_API_KEY_HERE` with your actual API key from your provider.

> **Note for Anthropic users:** Anthropic uses a different header format. Use `x-api-key` as the header name and your raw API key (without `Bearer`) as the value. You may need a separate credential for Anthropic's API.

### 4. Save the Credential

Click **Save** (or the equivalent button in your n8n version).

The credential is now stored in n8n's encrypted credential vault. It is not written to disk in plain text.

---

## Attaching the Credential to Workflow Nodes

After creating the credential, attach it to the two AI HTTP request nodes in the workflow:

### AI: Analyze Loop Node

1. Open the **Loop Leak Detector Automation** workflow.
2. Click the **AI: Analyze Loop** node.
3. In the node's **Authentication** dropdown, select **Predefined Credential Type**.
4. In the **Credential Type** dropdown, select **HTTP Header Auth**.
5. In the **Credential** dropdown, select the credential you just created.
6. Save the node.

### AI: Weekly Pattern Review Node

1. Click the **AI: Weekly Pattern Review** node.
2. Repeat steps 3–6 above.

---

## Updating the Provider Config Node

After attaching credentials, ensure the `credentialName` field in the **Provider Config** node matches the name of your credential exactly:

```javascript
const config = {
  apiUrl: 'https://api.openai.com/v1/chat/completions',
  model: 'gpt-4o',
  credentialName: 'OpenAI - Loop Leak Detector',  // ← must match exactly
  ...
};
```

Also update the `apiUrl` and `model` to match your provider.

---

## Switching Providers

To switch from one AI provider to another:

1. Create a new credential for the new provider following the steps above.
2. Update the **Provider Config** node:
   - Change `apiUrl` to the new provider's endpoint.
   - Change `model` to a model supported by the new provider.
   - Change `credentialName` to the new credential name.
3. Update the credential reference in both AI request nodes.
4. Test with a sample payload before activating in production.

Most OpenAI-compatible providers (Groq, Mistral, Together AI, Ollama) will work without any changes to the prompt structure. Anthropic requires a slightly different request format — adjust the HTTP request body in the **Prep Loop Analysis Request** node if switching to Anthropic's native API.

---

## Security Rules

1. **Never commit API keys to GitHub.** This repository's `.gitignore` should exclude any `.env` or secrets file. Do not work around this.
2. **Never paste API keys into workflow node parameters directly.** Always use n8n's credential vault.
3. **Never share workflow exports that contain real credential IDs.** The workflow JSON references credentials by ID — if exported after setup, scrub credential IDs before sharing.
4. **Rotate keys if exposed.** If a key is accidentally committed, rotate it immediately at your provider's dashboard.
5. **Use environment variables for n8n itself.** If self-hosting, configure n8n's `N8N_ENCRYPTION_KEY` environment variable to protect the credential vault.

---

## Verifying the Credential Works

After setup, send the sample payload from `schemas/activity-input-payload-example.json` to your webhook endpoint.

If you receive a 401 or 403 response from the AI request node:
- Verify the API key is valid and not expired.
- Verify the header format is correct (`Bearer KEY` for most providers).
- Verify the `apiUrl` is correct for your provider.
- Check your provider's usage dashboard to confirm the key is active.

If you receive a 429 (rate limit):
- Your account may be on a free tier or low usage limit.
- Upgrade your provider plan or wait for the rate limit to reset.
- The workflow already includes retry logic (3 retries with 2-second wait).
