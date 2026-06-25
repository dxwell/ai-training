# Connecting the AI Governance Lab to AWS Bedrock

## Why you'd do this

The lab currently calls `api.anthropic.com` directly, which requires an Anthropic API key. If your organisation runs everything inside AWS — for data residency, cost consolidation, or security boundary reasons — you can reroute the lab to call Claude through **Amazon Bedrock** instead. APRA CPG 234 may make this relevant: Bedrock keeps inference inside the AWS security boundary, with zero Anthropic personnel access to the infrastructure.

There are two Bedrock endpoint types. Choose based on your organisation's needs:

| | `bedrock-mantle` | `bedrock-runtime` |
|---|---|---|
| **API shape** | Native Messages API — nearly identical to `api.anthropic.com` | InvokeModel API — different request/response format |
| **Auth** | Bedrock API key (simple) or AWS SigV4 | AWS SigV4 required |
| **Code change** | Minimal — swap the URL and one header | Moderate — request body changes |
| **Recommended for** | This training lab | Backend server use cases |

**For this lab, use `bedrock-mantle`.** It requires the fewest code changes and runs from the browser the same way the direct API does.

---

## Prerequisites

Before touching the lab code, complete these steps in AWS.

### 1. AWS account with Bedrock access

You need an AWS account with Amazon Bedrock enabled. If your organisation already uses AWS, your cloud team will handle this.

### 2. Enable Claude model access

1. Go to the [AWS Console → Amazon Bedrock → Model access](https://console.aws.amazon.com/bedrock/home#/modelaccess)
2. Click **Modify model access**
3. Find **Anthropic → Claude Sonnet 4.6** and tick it
4. Submit the request — approval is typically instant for on-demand access

> **Australian data residency note:** Select `ap-southeast-2` (Sydney) as your region if you need data to stay in Australia. Check the [Bedrock regional availability table](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html) to confirm Claude Sonnet 4.6 is available there before proceeding.

### 3. Create a Bedrock API key

The `bedrock-mantle` endpoint supports API-key authentication — the same pattern the lab already uses.

1. Go to **AWS Console → Amazon Bedrock → API keys** (in the left nav)
2. Click **Create API key**
3. Give it a name: `ai-governance-lab-training`
4. Copy the key immediately — AWS will not show it again
5. Store it somewhere safe (a password manager, not a spreadsheet)

> This key authenticates against `bedrock-mantle` endpoints only. It is separate from your AWS access keys.

---

## Code change — Option A: bedrock-mantle (recommended)

This is the minimal change. The request body is identical to the current Anthropic API call. Only the URL and one header change.

### Find the `callClaude` function

In the lab HTML file, search for this function (around line 390):

```javascript
async function callClaude(systemPrompt, userContent, targetEl, style = '') {
  ...
  const resp = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      model: 'claude-sonnet-4-6',
      max_tokens: 1000,
      system: systemPrompt,
      messages: [{ role: 'user', content: userContent }]
    })
  });
```

### Replace it with this

```javascript
// ── BEDROCK CONFIGURATION ──
// Replace these two values before running the lab
const BEDROCK_API_KEY = 'YOUR_BEDROCK_API_KEY_HERE';
const BEDROCK_REGION  = 'ap-southeast-2';  // Sydney — change if using a different region

async function callClaude(systemPrompt, userContent, targetEl, style = '') {
  const el = document.getElementById(targetEl);
  el.className = 'response-box visible thinking' + (style ? ' ' + style : '');
  el.innerHTML = '<span class="spinner"></span> Calling Claude via AWS Bedrock...';

  try {
    const resp = await fetch(
      `https://bedrock-mantle.${BEDROCK_REGION}.api.aws/anthropic/v1/messages`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'x-api-key': BEDROCK_API_KEY,           // ← replaces the Anthropic API key
          'anthropic-version': '2023-06-01',        // ← required by bedrock-mantle
        },
        body: JSON.stringify({
          model: 'anthropic.claude-sonnet-4-6-v1',  // ← Bedrock model ID format
          max_tokens: 1000,
          system: systemPrompt,
          messages: [{ role: 'user', content: userContent }]
        })
      }
    );

    const data = await resp.json();
    if (data.error) throw new Error(data.error.message);
    const text = data.content.map(b => b.text || '').join('');
    el.className = 'response-box visible' + (style ? ' ' + style : '');
    el.textContent = text;
    return text;
  } catch (err) {
    el.className = 'response-box visible error';
    el.textContent = '⚠ Bedrock API Error: ' + err.message;
    return null;
  }
}
```

### What changed and why

| What changed | Old value | New value | Why |
|---|---|---|---|
| URL | `https://api.anthropic.com/v1/messages` | `https://bedrock-mantle.{region}.api.aws/anthropic/v1/messages` | Routes to AWS infrastructure |
| Auth header name | `x-api-key` | `x-api-key` | Same — but the key value is your Bedrock API key |
| Version header | *(not needed for direct API)* | `anthropic-version: 2023-06-01` | Required by bedrock-mantle |
| Model ID | `claude-sonnet-4-6` | `anthropic.claude-sonnet-4-6-v1` | Bedrock uses namespaced model IDs |

The request body (`messages`, `system`, `max_tokens`) and response shape (`data.content`) are **identical**. No other changes needed.

---

## Code change — Option B: bedrock-runtime with AWS SigV4

Use this if your organisation's security policy prohibits API-key authentication and requires SigV4 signing (IAM roles, temporary credentials). This is more complex and is typically done when the lab is served from a backend rather than opened as a local HTML file.

> **Practical note for training delivery:** SigV4 signing cannot be done safely in a browser without exposing AWS credentials. If you need SigV4, serve the lab from a lightweight backend (Lambda, EC2, or a local Node.js server) that signs requests server-side, or use Option A with a short-lived Bedrock API key for the training session.

The request body for `bedrock-runtime` also differs — it requires an `anthropic_version` field inside the body:

```javascript
body: JSON.stringify({
  anthropic_version: 'bedrock-2023-05-31',  // ← inside body, not a header
  max_tokens: 1000,
  system: systemPrompt,
  messages: [{ role: 'user', content: userContent }]
})
```

And the endpoint URL format is:
```
https://bedrock-runtime.{region}.amazonaws.com/model/{modelId}/invoke
```

For `claude-sonnet-4-6-v1` in Sydney:
```
https://bedrock-runtime.ap-southeast-2.amazonaws.com/model/anthropic.claude-sonnet-4-6-v1/invoke
```

---

## Region reference

| Location | AWS Region | bedrock-mantle endpoint |
|---|---|---|
| Sydney (recommended for AU data residency) | `ap-southeast-2` | `bedrock-mantle.ap-southeast-2.api.aws` |
| Singapore | `ap-southeast-1` | `bedrock-mantle.ap-southeast-1.api.aws` |
| US East (N. Virginia) | `us-east-1` | `bedrock-mantle.us-east-1.api.aws` |
| US West (Oregon) | `us-west-2` | `bedrock-mantle.us-west-2.api.aws` |

Verify that `anthropic.claude-sonnet-4-6-v1` is available in your chosen region in the [Bedrock model availability table](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html) before the training session.

---

## Testing before the session

Run this test in your browser console after making the change, replacing `YOUR_KEY` and `YOUR_REGION`:

```javascript
fetch('https://bedrock-mantle.YOUR_REGION.api.aws/anthropic/v1/messages', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-api-key': 'YOUR_BEDROCK_API_KEY',
    'anthropic-version': '2023-06-01'
  },
  body: JSON.stringify({
    model: 'anthropic.claude-sonnet-4-6-v1',
    max_tokens: 100,
    messages: [{ role: 'user', content: 'Reply with: Bedrock connection confirmed.' }]
  })
})
.then(r => r.json())
.then(d => console.log(d.content[0].text))
.catch(e => console.error(e));
```

Expected output: `Bedrock connection confirmed.`

If you see a CORS error, the lab needs to be served from a local web server rather than opened as a file (see next section).

---

## CORS consideration

AWS `bedrock-mantle` endpoints have CORS restrictions that may block browser-based `fetch` calls when the HTML file is opened directly from disk (`file://`). If you hit this during testing:

**Quick fix for training delivery:** Serve the file from a local web server rather than opening it directly.

```bash
# Python (no install needed — built into Python 3)
cd /path/to/your/training/folder
python3 -m http.server 8080
# Then open http://localhost:8080/ai-governance-lab.html
```

```bash
# Node.js alternative
npx serve .
# Then open the URL it shows
```

If your organisation uses a managed training environment (e.g., an internal web server or S3 static site), deploy the HTML file there and CORS should not be an issue.

---

## Security checklist before training day

- [ ] Bedrock API key is stored outside the HTML file (use an environment variable or enter it at session start)
- [ ] Key has only `bedrock:InvokeModel` permissions — not account-wide AWS access
- [ ] Key will be rotated or deleted after the training session
- [ ] Region is confirmed as `ap-southeast-2` if data must stay in Australia
- [ ] Claude Sonnet 4.6 model access has been approved in the AWS Console
- [ ] CORS test passed — lab loads and responds correctly from your serving environment
- [ ] Backup: direct Anthropic API key ready in case Bedrock is unreachable during the session

---

## If something goes wrong on the day

| Symptom | Likely cause | Fix |
|---|---|---|
| `403 Forbidden` | Model access not enabled, or wrong API key | Check Bedrock Model Access in AWS Console |
| `404 Not Found` | Wrong model ID or region | Confirm `anthropic.claude-sonnet-4-6-v1` exists in your region |
| CORS error in browser | File opened from disk, not a web server | Serve with `python3 -m http.server 8080` |
| `ValidationException` | Missing `anthropic-version` header | Ensure the header is in every fetch call |
| Timeout / no response | Region doesn't have capacity | Switch to `us-east-1` as fallback |

Keep the original Anthropic API key in your pocket as a fallback. Switching back takes 30 seconds — change the URL and key in the two constants at the top of `callClaude`.
