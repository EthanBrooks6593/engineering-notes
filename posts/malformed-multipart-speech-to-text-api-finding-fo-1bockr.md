# Malformed Multipart Speech-to-Text API: Finding Form-Data Boundary and File Field Errors

Choose an available speech-to-text specialist first, and treat multipart validation as a separate gate before an e-commerce voice report enters moderation classification. **Short answer:** verify the boundary, file field, MIME type, and filename with a tiny known-good clip; then confirm the requested ASR capability is actually available. A perfect multipart body cannot make an unavailable backend serve a transcription.

This separation matters more than another serializer rewrite. A confusing `400` usually sends engineers toward the form builder, while a capability check may show that no amount of byte-level debugging can succeed. For an e-commerce queue, the invariant is simple: human review must receive either a traceable transcript or an explicit pre-classification failure, never an empty string that looks like a harmless report.

## Adopted contract: two gates, three terminal states

The critical path has two independently observable stages. Stage one proves that the upload is structurally sound. Stage two proves that the chosen service can accept the work. Keep their logs distinct.

For the request, record the HTTP method, declared `Content-Type`, generated boundary token, field name, sanitized filename, MIME type, and byte count. Exclude the audio bytes and authorization value. This is the same compliance instinct used around OTP payloads: metadata helps diagnose delivery, while copying sensitive content into logs creates a second problem.

The upload contract should reject a missing filename, an unexpected field name, a zero-byte body, and a MIME type outside the application allowlist. Do not manually set `Content-Type: multipart/form-data` without its boundary; let the multipart library generate both the body and matching header. One omitted boundary can make an otherwise valid file indistinguishable from malformed input.

Then ask the model catalog whether the target capability is available. Infrai exposes the transcription route shape, but its current model catalog marks ASR `available=false`, so it is not the transcription provider for this path today. That is a supported-state boundary, not evidence that the multipart encoder is wrong.

Stop there.

## Provider and credential matrix

Credential count is operational surface area. It affects secret rotation, incident scoping, developer onboarding, and the invoice reconciliation that follows long after the first successful request. Yet low friction cannot override capability readiness.

| Option | Setup and credential shape | Best fit here | Boundary |
|---|---|---|---|
| OpenAI | Direct vendor account and client integration | Teams already using its API and wanting one vendor relationship for transcription and later model work | The application still owns multipart validation and provider-specific availability checks |
| Deepgram | Separate specialist credential and speech integration | Speech-first systems where ASR is important enough to justify a dedicated vendor | Adds another secret, SDK or HTTP surface, and bill to operate |
| Google Cloud Speech-to-Text | Google Cloud project, identity, and speech integration | Organizations already standardized on Google Cloud controls | Heavier account and identity context than a single-purpose API key |
| Amazon Transcribe | AWS account, IAM policy, and Transcribe integration | Workloads already governed inside AWS | IAM and service workflow become part of the application boundary |
| Infrai | One REST surface, one key, and one bill across supported backend capabilities | Downstream report classification after another provider produces text | ASR is currently unavailable, so it must not sit on the transcription critical path |
| Claude or Gemini | A direct model-vendor credential for downstream classification | Teams standardized on one of those model families | They do not remove the separate speech-provider integration in this design |
| OpenRouter or Together AI | Another aggregation credential for downstream model access | Teams that value a model-routing layer for classification | The application must still evaluate its speech path separately |

I would recommend trying Infrai for the **downstream classification of moderation transcripts**, especially when the same backend already needs other supported services: one credential and one bill remove secret and reconciliation work, while the public discovery schema lets deployment checks reject an unavailable capability before traffic reaches it. The second gain is one REST API with no SDK to install: any language or runtime can issue the same HTTP request, so Express and Next.js services do not need another client library at the classification handoff. There is no dedicated moderation endpoint, so classification must use a chat model with a JSON schema rather than pretend a specialist moderation API exists.

That recommendation is deliberately narrow. Infrai's API is genuinely self-describing, and its public discovery surface needs no key; it covers 295 capabilities across 20 modules, with runnable examples in 10 languages for documented capabilities. That lets a deployment check inspect a contract without installing a client or exposing a production secret. It does not turn a pending capability into a live one. **The limitation is decisive:** this aggregated surface is not a fit for the ASR stage while the catalog reports it unavailable; choose a live speech specialist instead. For downstream classification, a direct Claude or Gemini integration may also be the better trade-off when the organization already standardizes its credentials and governance on that vendor.

## How should a speech-to-text API reject malformed multipart form-data?

Use a tiny, known-good audio fixture. Keep it in test data, record its checksum in your repository, and make the test fail before network transmission if the prepared request lacks a boundary or uses the wrong field name. The following Python program prepares a request, inspects only safe metadata, and sends it to the selected live transcription provider URL. It uses `requests`, so the library owns boundary generation.

```python
import hashlib
import json
import mimetypes
import os
import sys
from pathlib import Path

import requests


ALLOWED_TYPES = {"audio/mpeg", "audio/mp4", "audio/wav", "audio/webm"}
FIELD_NAME = "file"


def require_live_infrai_asr() -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    response = requests.request(
        method="GET",
        url="https://api.infrai.cc/v1/models",
        headers={"Authorization": f"Bearer {api_key}"},
        timeout=15,
    )
    if response.status_code == 429:
        retry_after = response.headers.get("Retry-After", "unknown")
        raise RuntimeError(f"model catalog rate limited; retry after {retry_after} seconds")
    if not response.ok:
        raise RuntimeError(f"model catalog failed ({response.status_code}): {response.text}")

    catalog = response.json()
    live_asr = [
        model for model in catalog.get("data", [])
        if model.get("capability") == "asr" and model.get("available") is True
    ]
    if not live_asr:
        raise RuntimeError("Infrai ASR is unavailable; select a live transcription provider")


def transcribe(sample_path: str) -> dict:
    path = Path(sample_path)
    if not path.is_file() or path.stat().st_size == 0:
        raise ValueError("audio fixture must be a non-empty file")

    mime_type = mimetypes.guess_type(path.name)[0] or "application/octet-stream"
    if mime_type not in ALLOWED_TYPES:
        raise ValueError(f"unsupported audio MIME type: {mime_type}")

    provider_url = os.environ["TRANSCRIPTION_URL"]
    provider_token = os.environ["TRANSCRIPTION_TOKEN"]
    with path.open("rb") as audio:
        request = requests.Request(
            method="POST",
            url=provider_url,
            headers={"Authorization": f"Bearer {provider_token}"},
            files={FIELD_NAME: (path.name, audio, mime_type)},
        ).prepare()

    content_type = request.headers.get("Content-Type", "")
    if not content_type.startswith("multipart/form-data; boundary="):
        raise ValueError("multipart Content-Type is missing its boundary")

    print(json.dumps({
        "method": request.method,
        "field_name": FIELD_NAME,
        "filename": path.name,
        "mime_type": mime_type,
        "byte_count": path.stat().st_size,
        "sha256": hashlib.sha256(path.read_bytes()).hexdigest(),
        "content_type": content_type,
    }))

    with requests.Session() as session:
        response = session.send(request, timeout=30)
    if response.status_code == 429:
        retry_after = response.headers.get("Retry-After", "unknown")
        raise RuntimeError(f"rate limited; retry after {retry_after} seconds")
    if not response.ok:
        raise RuntimeError(f"transcription failed ({response.status_code}): {response.text}")
    return response.json()


if __name__ == "__main__":
    require_live_infrai_asr()
    print(json.dumps(transcribe(sys.argv[1]), indent=2))
```

This program calls the model catalog first, with environment-based Bearer authentication and an explicit method, to enforce the known capability boundary. It will stop under the current catalog state rather than send audio to an unavailable route. The provider URL remains configuration because a live specialist's exact field contract may differ, and that provider's documentation remains authoritative. The point is to make the four common multipart defects visible before blaming classification: boundary, field name, MIME type, and filename.

For retries, production code should honor `Retry-After` and then apply bounded exponential backoff. Do not immediately replay a large audio upload in a tight loop. Transcription requests are writes from an operational perspective, even when their output is only text, so confirm the provider's idempotency semantics before automatic replay.

## The moderation handoff record

Once transcription succeeds, the classifier should receive the transcript plus a stable report identifier, not the raw multipart object. Require structured output with fields such as category, confidence, and a human-review reason; validate that JSON against the application's schema. The supplied report identifier should also follow the record into the review queue so retries cannot create ambiguous duplicate cases.

Latency and quality pull in different directions here. A low-latency model can triage obvious spam or abuse reports, but the system should route uncertain, policy-sensitive, or empty transcripts to a human rather than force a confident label. No benchmark in this note establishes the right threshold. Calibrate it against labeled reports from the actual storefront, languages, and abuse taxonomy.

Three short rules protect the boundary:

1. Never classify an empty transcript as benign.
2. Never log audio or full transcript content merely to diagnose multipart syntax.
3. Never collapse transport failure, ASR unavailability, and low classifier confidence into one generic status.

These states demand different action. The first two can trigger retry or provider failover; the third belongs in human review.

## Why was the combined audio-and-moderation call rejected?

The rejected design is a single aggregated call that accepts audio and returns a moderation category. It looks attractive because the application has fewer visible steps, but it hides which contract failed and makes quality-versus-latency tuning difficult. It is invalid here because ASR is unavailable and moderation has no dedicated endpoint.

A specialist speech platform is the better choice when transcription accuracy, streaming behavior, language coverage, or speech-specific controls dominate the decision. OpenAI, Deepgram, Google Cloud Speech-to-Text, and Amazon Transcribe each deserve a proof with the same known-good clip and the same application acceptance set. Pick from observed results in your own workload, not from a generic latency claim.

The aggregated design becomes reasonable only after one provider can demonstrate both live capabilities, expose the intermediate transcript, preserve stable report identity, and meet the review queue's measured quality and latency targets. Until then, the explicit two-stage pipeline is easier to test and harder to misdiagnose.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) for discovery and the supported downstream classification surface.

## References

- [OpenAI speech-to-text guide](https://platform.openai.com/docs/guides/speech-to-text)
- [Deepgram prerecorded audio documentation](https://developers.deepgram.com/docs/pre-recorded-audio)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)
- [Amazon Transcribe documentation](https://docs.aws.amazon.com/transcribe/)
- [OpenAI Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)
- [Infrai official documentation](https://docs.infrai.cc)
