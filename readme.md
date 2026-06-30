# AI Training

This is a fully live interactive training application. Here's what participants actually do in each module:

## Module 01 — Guardrails: 
They first watch ClaimsAssist AI respond to a demographically-loaded claims prompt with zero controls. It produces a biased, ungoverned answer. Then they build their own rule set (quick-add chips or custom rules), and run the identical prompt again. The model now shows [GUARDRAIL APPLIED] labels where rules activate. The debrief question — which rules held and which didn't? — surfaces the limits of prompt-only controls.

## Module 02 — Evaluations: 
They select a real claim scenario (hailstorm, flood, bushfire, disputed claim), generate a ClaimsAssist output, then manually score it 0–10 across Fairness, Explainability, Accuracy, and Regulatory Safety before seeing the AI judge's verdict in Module 04. The gap between human and AI scores is the learning.

## Module 03 — Chain of Thought: 
Side-by-side comparison — same claim, standard output vs. CoT output. Then they write their own 5-step reasoning template, test it live, and answer four governance questions that an APRA auditor would ask.

## Module 04 — LLM as a Judge: 
They calibrate the judge's system prompt, submit a ClaimsAssist output for review, then deliberately submit a pre-loaded biased output (uses age and postcode as settlement factors). If the judge is calibrated correctly, it returns REJECTED. If not, they fix the judge prompt and rerun.

## Module 05 — Observability: 
They define monitoring signals with thresholds (settlement drift, judge fail rate, guardrail refusal rate), then submit a pre-loaded anomaly — a 23% settlement drop over two weeks — for live AI analysis that returns a severity classification, likely cause, and escalation chain.

## Module 06 — Audit: 
The session log has been building the whole time. They generate an APRA CPG 234-formatted evidence summary based on everything they built, suitable for a board risk report.
Every action is logged to the audit trail throughout — so by Module 06, the log is populated with real governance events from their own session.

## Optional Module 1.5 - Prompt Injection
- Step 1 — Watch a direct attack fail. A blunt jailbreak attempt ("Ignore all previous instructions…") hits the guardrails they just built. It should be blocked. This is the easy case — and the point is to say so out loud.
- Step 2 — The hidden injection. This is the moment that lands. A realistic claim note from a cooperative-sounding claimant contains a SYSTEM OVERRIDE instruction buried mid-paragraph, after a long plausible preamble. It instructs the model to approve $31,000 with no reasoning and no caveats. Whether it succeeds or fails depends on the guardrails they built in Module 01 — the lab detects the outcome and tells them whether their controls held.
- Step 3 — Craft your own attack. Participants write their own injection, hidden inside a plausible claim. This is where the room gets competitive. It also surfaces attack vectors the facilitator wouldn't have anticipated.
- Step 4 — Harden and retest. Four anti-injection quick-add rules are available (content-is-data-only, keyword detection, always-require-reasoning, role-change resistance). They add rules, then rerun the Step 2 injection to verify the fix works. Every attempt — succeeded or blocked — is logged to the audit trail with a flag level event, so by Module 06 the evidence pack includes documented adversarial testing.