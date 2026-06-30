# Deploying the AI Governance Lab to AWS — S3 + Lambda + Bedrock

## Architecture overview

This setup uses three AWS services together, not as alternatives:

```
┌─────────────┐      ┌──────────────────────┐      ┌─────────────────┐
│   S3        │      │   Lambda Function URL │      │  Amazon Bedrock │
│  (static    │ ───▶ │   (proxy + SigV4      │ ───▶ │  Claude Sonnet  │
│   website)  │      │    signing handled     │      │   4.6           │
│             │      │    by IAM role)         │      │                 │
└─────────────┘      └──────────────────────┘      └─────────────────┘
   Hosts the              Receives browser              Generates the
   HTML/JS file           fetch() calls,                 model response
   participants            calls Bedrock                  and returns it
   open in their           server-side
   browser
```

**Why you need both S3 and Lambda — not one or the other.**

S3 hosts the static HTML file so participants can reach it via a URL instead of a local file. But the browser cannot safely call Bedrock directly: Bedrock requires AWS Signature Version 4 (SigV4) request signing, and signing requires AWS credentials that must never be exposed in client-side JavaScript. Lambda solves this by sitting between the browser and Bedrock — it holds the AWS credentials (via an IAM execution role, never a hardcoded key) and does the signing on the server side. The browser only ever talks to the Lambda Function URL over plain HTTPS with no AWS credentials involved.

This is the same proxy pattern used by every production "AI gateway" architecture — the Lambda function is the trust boundary.

---

## Part 1 — Set up the Lambda proxy function

### Step 1.1 — Enable Bedrock model access

If you haven't already:

1. Go to **AWS Console → Amazon Bedrock → Model access**
2. Click **Modify model access**, enable **Anthropic → Claude Sonnet 4.6**
3. Submit — approval is typically instant

> **Australian data residency:** Choose `ap-southeast-2` (Sydney) as your working region throughout this guide if data must stay in Australia. Confirm Claude Sonnet 4.6 is available there in the [Bedrock regional availability table](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html) before continuing.

### Step 1.2 — Create the Lambda function

1. Go to **AWS Console → Lambda → Create function**
2. Choose **Author from scratch**
3. Function name: `ai-governance-lab-proxy`
4. Runtime: **Python 3.12**
5. Architecture: `x86_64`
6. Leave the default execution role — you'll add Bedrock permissions next
7. Click **Create function**

### Step 1.3 — Write the proxy code

Replace the default code in the inline editor (`lambda_function.py`) with the following:

```python
import json
import boto3
import os

bedrock_runtime = boto3.client(
    "bedrock-runtime",
    region_name=os.environ.get("BEDROCK_REGION", "ap-southeast-2")
)

MODEL_ID = os.environ.get("BEDROCK_MODEL_ID", "anthropic.claude-sonnet-4-6-v1")

# CORS headers — the lab's S3 site origin goes here once you know it
CORS_HEADERS = {
    "Access-Control-Allow-Origin": "*",  # tighten this after deployment — see Part 3
    "Access-Control-Allow-Headers": "Content-Type",
    "Access-Control-Allow-Methods": "POST, OPTIONS",
}


def lambda_handler(event, context):
    # Handle CORS preflight
    method = event.get("requestContext", {}).get("http", {}).get("method", "")
    if method == "OPTIONS":
        return {"statusCode": 200, "headers": CORS_HEADERS, "body": ""}

    try:
        body = json.loads(event.get("body", "{}"))

        system_prompt = body.get("system", "")
        messages = body.get("messages", [])
        max_tokens = body.get("max_tokens", 1000)

        bedrock_payload = {
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": max_tokens,
            "messages": messages,
        }
        if system_prompt:
            bedrock_payload["system"] = system_prompt

        response = bedrock_runtime.invoke_model(
            modelId=MODEL_ID,
            body=json.dumps(bedrock_payload),
            contentType="application/json",
            accept="application/json",
        )

        response_body = json.loads(response["body"].read())

        return {
            "statusCode": 200,
            "headers": {**CORS_HEADERS, "Content-Type": "application/json"},
            "body": json.dumps(response_body),
        }

    except Exception as e:
        return {
            "statusCode": 500,
            "headers": {**CORS_HEADERS, "Content-Type": "application/json"},
            "body": json.dumps({"error": {"message": str(e)}}),
        }
```

Click **Deploy** to save.

### Step 1.4 — Configure environment variables

In the Lambda console, go to **Configuration → Environment variables → Edit**, add:

| Key | Value |
|---|---|
| `BEDROCK_REGION` | `ap-southeast-2` |
| `BEDROCK_MODEL_ID` | `anthropic.claude-sonnet-4-6-v1` |

### Step 1.5 — Increase the timeout

Default Lambda timeout (3 seconds) is too short for an LLM call.

1. **Configuration → General configuration → Edit**
2. Set **Timeout** to `30 sec`
3. Set **Memory** to `512 MB` (plenty for this proxy workload)
4. Save

### Step 1.6 — Grant the function permission to call Bedrock

1. **Configuration → Permissions** — click the execution role link (opens IAM)
2. **Add permissions → Create inline policy**
3. Switch to the **JSON** tab and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowBedrockInvoke",
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": "arn:aws:bedrock:ap-southeast-2::foundation-model/anthropic.claude-sonnet-4-6-v1"
    }
  ]
}
```

4. Name it `BedrockInvokePolicy`, click **Create policy**

> Scoping the `Resource` to the specific model ARN (rather than `*`) means this Lambda function can never be used to call any other model or AWS service — useful if anyone asks how you've limited blast radius for the training session.

### Step 1.7 — Create the Function URL

1. Back in the Lambda console, go to **Configuration → Function URL → Create function URL**
2. **Auth type:** `NONE` — this lab is for an in-room training session with no sensitive production data, so public access without IAM signing keeps the browser code simple. (See **Part 4 — Security hardening** for tightening this.)
3. **Configure CORS:** tick this box now
4. Click **Save**
5. Copy the generated **Function URL** — it looks like:
   `https://abc123xyz456.lambda-url.ap-southeast-2.on.aws/`

You now have a working HTTPS endpoint that accepts a POST request and returns a Claude response, with no AWS credentials anywhere near the browser.

### Step 1.8 — Test the Lambda function directly

```bash
curl -X POST https://abc123xyz456.lambda-url.ap-southeast-2.on.aws/ \
  -H "Content-Type: application/json" \
  -d '{
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "Reply with: Bedrock proxy confirmed."}]
  }'
```

Expected response includes `"text": "Bedrock proxy confirmed."` inside the JSON body.

If you get a `403` or `AccessDeniedException`, double check Step 1.6 — the IAM policy is the most common point of failure.

---

## Part 2 — Update the lab's HTML to call the Lambda proxy

Open `ai-governance-lab.html` and find the `callClaude` function (search for `async function callClaude`).

### Replace the fetch call

```javascript
// ── LAMBDA PROXY CONFIGURATION ──
// Paste your Function URL from Step 1.7 here
const LAMBDA_PROXY_URL = 'https://abc123xyz456.lambda-url.ap-southeast-2.on.aws/';

async function callClaude(systemPrompt, userContent, targetEl, style = '') {
  const el = document.getElementById(targetEl);
  el.className = 'response-box visible thinking' + (style ? ' ' + style : '');
  el.innerHTML = '<span class="spinner"></span> Calling Claude via Bedrock...';

  try {
    const resp = await fetch(LAMBDA_PROXY_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        max_tokens: 1000,
        system: systemPrompt,
        messages: [{ role: 'user', content: userContent }]
      })
    });

    const data = await resp.json();
    if (data.error) throw new Error(data.error.message);
    const text = data.content.map(b => b.text || '').join('');
    el.className = 'response-box visible' + (style ? ' ' + style : '');
    el.textContent = text;
    return text;
  } catch (err) {
    el.className = 'response-box visible error';
    el.textContent = '⚠ Proxy Error: ' + err.message;
    return null;
  }
}
```

### Note on response shape

The Lambda function returns the raw Bedrock InvokeModel response, which — for Claude on Bedrock — has the same `content: [{ type, text }]` shape as the direct Anthropic API, because Bedrock's native Claude integration mirrors the Messages API format. No additional parsing changes are needed in the rest of the lab's JavaScript (the `generateForJudge` function, which also calls the API directly, needs the same URL swap — see Part 2.1 below).

### Step 2.1 — Update the second fetch call

The lab has one more place that calls the API directly: the `generateForJudge` function in Module 04. Find it and apply the same change:

```javascript
async function generateForJudge() {
  const claim = scenarios.hail;
  const sys = buildGuardrailSystem();
  logAudit('JUDGE_INPUT_GENERATED', 'Fresh ClaimsAssist output generated for judge review', 'info');
  const el = document.getElementById('judge-input');
  el.value = 'Generating...';
  try {
    const resp = await fetch(LAMBDA_PROXY_URL, {   // ← changed from api.anthropic.com
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ max_tokens: 1000, system: sys, messages: [{ role: 'user', content: claim }] })
    });
    const data = await resp.json();
    el.value = data.content.map(b => b.text || '').join('');
  } catch(e) { el.value = 'Error generating output.'; }
}
```

---

## Part 3 — Host the lab on S3

### Step 3.1 — Create the bucket

1. **AWS Console → S3 → Create bucket**
2. Bucket name: `ai-governance-lab-training` (must be globally unique — add a suffix if taken, e.g. `ai-governance-lab-training-meridian`)
3. Region: same region as your Lambda function (e.g. `ap-southeast-2`)
4. **Uncheck "Block all public access"** — you'll see a warning; acknowledge it (this is a public training asset, not sensitive data — see Part 4 if you'd rather keep it private)
5. Leave other settings at default, click **Create bucket**

### Step 3.2 — Upload the lab file

1. Open the bucket, click **Upload**
2. Add your edited `ai-governance-lab.html` (with the Lambda proxy URL already in place from Part 2)
3. Click **Upload**

### Step 3.3 — Enable static website hosting

1. In the bucket, go to **Properties → Static website hosting → Edit**
2. Select **Enable**
3. **Index document:** `ai-governance-lab.html`
4. Save

### Step 3.4 — Add a bucket policy for public read access

1. Go to **Permissions → Bucket policy → Edit**
2. Paste (replace `YOUR-BUCKET-NAME`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```

3. Save changes

### Step 3.5 — Get your training URL

Go back to **Properties → Static website hosting** — copy the **Bucket website endpoint**. It looks like:

```
http://ai-governance-lab-training.s3-website-ap-southeast-2.amazonaws.com
```

This is the URL participants open in their browsers on training day.

### Step 3.6 — Lock down CORS on the Lambda side (recommended)

Now that you know the S3 site's origin, go back to your Lambda function code (Step 1.3) and tighten the CORS header from `"*"` to your actual S3 endpoint:

```python
CORS_HEADERS = {
    "Access-Control-Allow-Origin": "http://ai-governance-lab-training.s3-website-ap-southeast-2.amazonaws.com",
    "Access-Control-Allow-Headers": "Content-Type",
    "Access-Control-Allow-Methods": "POST, OPTIONS",
}
```

Redeploy the Lambda function after this change.

---

## Part 4 — Security hardening (recommended before a real training session)

The setup above prioritises simplicity for a training-room context. For anything beyond a one-off internal session, consider:

| Risk | Hardening option |
|---|---|
| Lambda Function URL is public with no auth | Add a shared secret header check inside the Lambda code (compare against an environment variable) and have the HTML send it — cheap, effective for a single session |
| Anyone could call your Lambda and run up Bedrock costs | Add **Lambda reserved concurrency** (e.g. limit to 5) to cap simultaneous calls during the session |
| S3 bucket is fully public | Use **CloudFront + Origin Access Control** instead of public bucket policy, and restrict via CloudFront signed URLs or a simple Lambda@Edge auth check |
| No usage visibility | Enable **CloudWatch Logs** on the Lambda function (on by default) to review session activity afterward — this also feeds your Module 06 audit trail narrative nicely |
| Model ID gets out of date | Don't hardcode `anthropic.claude-sonnet-4-6-v1` in the HTML — it's already isolated to the Lambda environment variable, so updates only happen in one place |

A simple shared-secret approach for a single training session:

```python
# Add to lambda_function.py
EXPECTED_SECRET = os.environ.get("LAB_SHARED_SECRET", "")

def lambda_handler(event, context):
    ...
    headers = event.get("headers", {})
    if EXPECTED_SECRET and headers.get("x-lab-secret") != EXPECTED_SECRET:
        return {"statusCode": 403, "headers": CORS_HEADERS, "body": json.dumps({"error": {"message": "Unauthorized"}})}
    ...
```

```javascript
// Add to the HTML fetch call
headers: { 'Content-Type': 'application/json', 'x-lab-secret': 'YOUR_SESSION_SECRET' }
```

This is not bulletproof (the secret is visible in page source) but stops casual discovery of the URL from being usable.

---

## Part 5 — Testing before training day

### 5.1 — End-to-end test

1. Open the S3 website URL in a browser
2. Go to **Module 01 — Guardrails**
3. Click **▶ Run Without Guardrails**
4. Confirm you see a live, generated response within a few seconds

### 5.2 — Check the browser console

Open DevTools (F12) → Console tab. If you see CORS errors:
- Confirm the Lambda's `Access-Control-Allow-Origin` matches your S3 endpoint exactly (including `http://` vs `https://`)
- Confirm you redeployed the Lambda function after any CORS header change

### 5.3 — Load test (optional, recommended for 20+ participants)

Run several `curl` requests in parallel to confirm the Lambda function scales without throttling:

```bash
for i in {1..10}; do
  curl -s -X POST https://YOUR-LAMBDA-URL/ \
    -H "Content-Type: application/json" \
    -d '{"max_tokens": 50, "messages": [{"role": "user", "content": "Say OK"}]}' &
done
wait
```

All 10 should return successfully. If any fail, check **Lambda → Configuration → Concurrency** isn't set lower than your expected simultaneous participant count.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| CORS error in browser console | Origin mismatch or stale Lambda deploy | Recheck `Access-Control-Allow-Origin`, redeploy Lambda |
| `403 Forbidden` from S3 | Bucket policy not applied, or "Block public access" still on | Recheck Step 3.1 and 3.4 |
| `403 AccessDeniedException` from Lambda | IAM policy missing or wrong resource ARN | Recheck Step 1.6, confirm model ARN matches your region |
| `ValidationException` from Bedrock | Missing `anthropic_version` in payload | Confirm `lambda_function.py` includes `"anthropic_version": "bedrock-2023-05-31"` |
| Lambda times out | Default 3-second timeout still in place | Recheck Step 1.5 |
| S3 page loads but shows blank/broken UI | Wrong index document name | Confirm Step 3.3 index document matches your uploaded filename exactly |
| Works for you, fails for participants | CORS locked to your IP/testing origin only, or Lambda concurrency too low | Confirm `Access-Control-Allow-Origin` is the S3 site, not `localhost`; raise concurrency limit |

---

## Pre-training-day checklist

- [ ] Bedrock model access approved for Claude Sonnet 4.6 in your chosen region
- [ ] Lambda function deployed, timeout set to 30s, memory 512MB
- [ ] Lambda IAM role has `bedrock:InvokeModel` scoped to the correct model ARN
- [ ] Lambda Function URL created and tested via `curl`
- [ ] `ai-governance-lab.html` updated with the Lambda proxy URL in both fetch locations (`callClaude` and `generateForJudge`)
- [ ] S3 bucket created, file uploaded, static website hosting enabled
- [ ] S3 bucket policy allows public `GetObject`
- [ ] Lambda CORS origin tightened to match the S3 website endpoint
- [ ] End-to-end test passed from a browser (not just `curl`)
- [ ] Load test passed for expected participant count
- [ ] Backup plan ready: direct Anthropic API key, in case AWS access drops mid-session
