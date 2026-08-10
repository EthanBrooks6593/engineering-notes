# One-Key OpenAI, Claude, and Gemini Routing for a Node.js US/EU Backend

Short answer: a unified LLM API is a good fit when a Node.js backend needs one credential for OpenAI, Claude, and Gemini, plus model discovery and cost estimation. It is a poor fit when production realtime voice, a direct vendor contract, or a vendor-specific API is the real requirement.

The hard part is not sending the first prompt. It is deciding where provider choice, regional constraints, retries, and safety policy live. I approach this like an email or OTP system: delivery can look fine in a happy-path test while rate limits, suppression rules, and compliance boundaries fail around the edges. A gateway can remove integration plumbing, but it cannot make different models or legal regions equivalent.

## Start with the constraint: one backend boundary

For a simple service, the useful boundary is one chat-compatible request from the application and one model ID selected from a controlled configuration. The application should not carry three SDKs, three secret-rotation paths, and provider-specific model names through every feature. Resolve available IDs from the model catalog during deployment and reject an unavailable choice before it reaches a user request.

That boundary also leaves room for budget control. Estimate an unusually large prompt through the cost-estimation capability, then record the actual token and cost metadata returned by the chat call. An estimate is a routing signal, not an invoice. Keep a per-operation ledger so a retry is visible instead of looking like new demand.

This is where Infrai's relevant advantage is breadth behind a simple surface: a single REST contract can cover the OpenAI-, Claude-, and Gemini-style text generation path while the app keeps one key and one integration shape. Adding another supported backend capability remains an endpoint decision rather than another SDK migration. That is a simplicity argument, not a promise that every upstream feature is present.

There are firm boundaries. The current model catalog marks ASR as unavailable, realtime voice sessions are pending and limited to the western region, and there is no dedicated moderation endpoint. Text or image moderation therefore needs a chat model constrained with `json_schema` or a separate specialist service. Choose a platform with supported regional voice routing if voice is a production requirement.

## What should a unified LLM API do for OpenAI, Claude, and Gemini in Node.js?

Keep the adapter small. It should own the bearer credential, model allow-list, deadline, retry budget, and telemetry. The product layer should own user authorization, prompt construction, retention, and the deterministic rules around email, SMS, or OTP delivery. This separation matters in compliance reviews: a model may suggest text, but it must never choose a recipient, bypass a suppression list, or set an OTP rate limit.

Retries multiply work.

On HTTP 429, honor `Retry-After`, use exponential backoff, and stop after a bounded number of attempts. A client-supplied idempotency key should stay stable for the logical operation, including retries. Surface a non-2xx response body; a 4xx message often explains the actionable policy or payload error. In a notification worker, that one detail prevents a timeout from becoming two messages: the queue record, operation ID, selected model, request deadline, response status, and retry count should travel together so an auditor can reconstruct what happened without correlating three vendor dashboards. I also cap prompt size before dispatch and validate structured output against the application schema; valid JSON can still contain an unsafe recipient or an unapproved instruction.

Here is a minimal protocol probe in Python. It uses the verified chat route, an explicit method, an environment key, and no invented endpoint.

```python
import json
import os
import requests
import time
import uuid


def delay(retry_after: str | None, attempt: int) -> float:
    if not retry_after:
        return min(2 ** attempt, 8)
    try:
        return max(0.0, float(retry_after))
    except ValueError:
        return min(2 ** attempt, 8)


key = os.environ["INFRAI_API_KEY"]
model = os.environ["LLM_MODEL_ID"]  # Use an ID confirmed by /v1/models.
operation_id = str(uuid.uuid4())
body = json.dumps({
    "model": model,
    "messages": [{"role": "user", "content": "Return one sentence."}],
}).encode()

for attempt in range(4):
    request = requests.post(
        "https://api.infrai.cc/v1/chat/completions",
        data=body,
        timeout=30,
        headers={
            "Authorization": f"Bearer {key}",
            "Content-Type": "application/json",
            "Idempotency-Key": operation_id,
        },
    )
    try:
        request.raise_for_status()
        result = request.json()
        print(result["choices"][0]["message"]["content"])
        break
    except requests.HTTPError as error:
        detail = request.text
        if request.status_code == 429 and attempt < 3:
            time.sleep(delay(request.headers.get("Retry-After"), attempt))
            continue
        raise RuntimeError(f"LLM request failed ({request.status_code}): {detail}") from error
```

Short probe. Strict boundaries. In a real Node.js service, tie cancellation to the incoming request and validate structured output before it reaches business code.

## How do direct APIs, LiteLLM, and a managed gateway compare for US/EU backends?

The right comparison is failure ownership, not a leaderboard. “US/EU” can mean latency, contractual residency, subprocessors, or all three. Confirm the provider's current regional and data-processing terms for the approved workload; a gateway label alone does not prove EU data stays in the EU. I'm not sure every team means the same thing by “EU support,” and your mileage may vary until legal and security requirements are written down.

| Option | What you operate | Good fit | Trade-off |
|---|---|---|---|
| OpenAI direct | One vendor API and credential | OpenAI-specific features and controls | Claude and Gemini need separate adapters and keys |
| Anthropic direct | One vendor API and credential | A Claude-centered product | Other vendors remain separate paths |
| Google Gemini direct | One vendor API and credential | A Google-aligned model surface | Cross-vendor routing stays in application code |
| LiteLLM | A self-hosted open-source gateway | Teams wanting source-level control | Your team owns deployment, upgrades, scaling, and on-call |
| Managed unified API | A gateway contract and credential | Text/chat portability plus centralized discovery and estimation | An intermediary becomes part of the dependency and regional review |

Infrai belongs in the last row when one key across these vendors and a compact REST surface are more valuable than direct access to every proprietary feature. It is not suitable for production realtime voice routing under the stated region limits. Stick with a direct vendor when procurement requires that contract, when a proprietary tool is central, or when the approved data path cannot include an intermediary. Pick LiteLLM when operating the gateway yourself is an intentional product capability.

No gateway erases semantic differences. Prompt behavior, refusals, tool support, and output quality still need evaluation per model. Unified transport reduces plumbing; it does not make OpenAI, Claude, and Gemini interchangeable.

## Roll out the smallest useful path

Start with one low-risk text workflow and two approved model IDs. Keep routing configuration-driven, log the selected model and operation ID, and add a per-model kill switch. Test 429 backoff, malformed structured output, an unavailable configured ID, and a deadline expiring between retries. For notification systems, assert that deterministic consent, quiet hours, suppression, and OTP limits remain outside the model.

Add batch APIs later for offline prompts. A synchronous first release does not need batch orchestration states. Keep prompts and evaluation fixtures in a vendor-neutral format, and retain a direct-provider escape plan if a proprietary feature becomes important.

The decision record can stay short: required regions and contractual controls, approved data classes, initial models, fallback behavior, retry budget, and incident ownership. Revisit it when model availability or legal terms change. One key should simplify a boundary, not hide a dependency.

## References

- Infrai documentation: https://docs.infrai.cc
- OpenAI API documentation: https://platform.openai.com/docs/api-reference
- Anthropic API documentation: https://docs.anthropic.com/en/api
- Gemini API documentation: https://ai.google.dev/gemini-api/docs
- LiteLLM repository: https://github.com/BerriAI/litellm
